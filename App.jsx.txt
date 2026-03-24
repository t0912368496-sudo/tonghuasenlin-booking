import React, { useState, useRef, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { 
  getFirestore, doc, setDoc, getDoc, collection, onSnapshot, query, addDoc, updateDoc, deleteDoc 
} from 'firebase/firestore';
import { 
  Calendar, Clock, User, Phone, Palette, CheckCircle2, ChevronRight, Heart, Camera, Settings, 
  Plus, X, CalendarDays, ListOrdered, LayoutGrid, Trash2, ChevronDown, ChevronUp, Users, 
  CreditCard, Wallet, AlertCircle, Ticket, Calendar as CalendarIcon 
} from 'lucide-react';

// --- Firebase 初始化配置 ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'fairy-forest-booking-v1';

const App = () => {
  // --- 狀態管理 ---
  const [user, setUser] = useState(null);
  const [isAdmin, setIsAdmin] = useState(false);
  const [step, setStep] = useState(1);
  const [expandedSlot, setExpandedSlot] = useState(null); 
  const [loading, setLoading] = useState(true);
  const fileInputRef = useRef(null);
  
  // 核心資料狀態
  const [bankInfo, setBankInfo] = useState({ code: '822', account: '123-456789-000', name: '童話森林工作室' });
  const [projects, setProjects] = useState([]);
  const [managedDates, setManagedDates] = useState({});
  const [bookings, setBookings] = useState([]);
  const [logoUrl, setLogoUrl] = useState("https://api.screenshotmachine.com/?key=ca8281&url=https%3A%2F%2Fstorage.googleapis.com%2Fstatic.invideo.io%2Fproduction%2F1711311025514_Screenshot_2026-03-24_at_11.27.50_PM.png&dimension=1024x768");

  // 表單輸入狀態
  const [newProj, setNewProj] = useState({ name: '', type: 'regular', date: '', timeSlot: '', price: '', points: '' });
  const [bookingForm, setBookingForm] = useState({ project: null, date: '', time: '', name: '', phone: '', payMethod: 'transfer', lastFive: '' });

  const times = ['09:00', '10:00', '11:00', '12:00', '13:00', '14:00', '15:00', '16:00', '17:00', '18:00'];

  // --- 權限與認證邏輯 ---
  useEffect(() => {
    // 規則 3：務必先進行身份驗證，並使用指數退避進行重試
    const loginWithRetry = async (retryCount = 0) => {
      try {
        await signInAnonymously(auth);
      } catch (err) {
        if (retryCount < 5) {
          const delay = Math.pow(2, retryCount) * 1000;
          setTimeout(() => loginWithRetry(retryCount + 1), delay);
        }
      }
    };

    loginWithRetry();
    
    // 監聽認證狀態
    const unsubscribeAuth = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      if (!currentUser) {
        setLoading(true); // 若失去登入狀態，顯示載入中
      }
    });

    return () => unsubscribeAuth();
  }, []);

  // --- Firestore 資料同步邏輯 ---
  useEffect(() => {
    // 規則 3：確保 user 存在後才發起查詢
    if (!user) return;

    // 定義資料路徑 (遵循規則 1)
    const settingsRef = doc(db, 'artifacts', appId, 'public', 'settings');
    const projectsCol = collection(db, 'artifacts', appId, 'public', 'data', 'projects');
    const bookingsCol = collection(db, 'artifacts', appId, 'public', 'data', 'bookings');

    // 1. 監聽全域設定
    const unsubSettings = onSnapshot(settingsRef, (docSnap) => {
      if (docSnap.exists()) {
        const data = docSnap.data();
        if (data.bankInfo) setBankInfo(data.bankInfo);
        if (data.managedDates) setManagedDates(data.managedDates);
        if (data.logoUrl) setLogoUrl(data.logoUrl);
      } else {
        // 第一次啟動時初始化
        initializeSettings();
      }
    }, (err) => {
      console.error("Settings Sync Error:", err);
    });

    // 2. 監聽課程清單
    const unsubProjects = onSnapshot(projectsCol, (snap) => {
      const projs = snap.docs.map(d => ({ id: d.id, ...d.data() }));
      setProjects(projs.length > 0 ? projs : [
        { id: 'default1', name: '拼豆', type: 'regular', price: 0, points: 0 },
        { id: 'default2', name: '羊毛氈', type: 'regular', price: 0, points: 0 }
      ]);
      setLoading(false);
    }, (err) => console.error("Projects Sync Error:", err));

    // 3. 監聽預約清單
    const unsubBookings = onSnapshot(bookingsCol, (snap) => {
      const bks = snap.docs.map(d => ({ id: d.id, ...d.data() }));
      setBookings(bks);
    }, (err) => console.error("Bookings Sync Error:", err));

    return () => {
      unsubSettings();
      unsubProjects();
      unsubBookings();
    };
  }, [user]);

  const initializeSettings = async () => {
    if (!user) return;
    const dates = {};
    const today = new Date();
    const endOfMonth = new Date(today.getFullYear(), today.getMonth() + 2, 0);
    for (let d = new Date(today); d <= endOfMonth; d.setDate(d.getDate() + 1)) {
      const dateStr = d.toISOString().split('T')[0];
      dates[dateStr] = (d.getDay() === 0 || d.getDay() === 6);
    }
    await setDoc(doc(db, 'artifacts', appId, 'public', 'settings'), { 
      managedDates: dates, 
      bankInfo, 
      logoUrl,
      updatedAt: new Date().toISOString()
    });
  };

  // --- 資料操作 ---

  const saveSettings = async (newData) => {
    if (!user) return;
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'settings'), newData, { merge: true });
    } catch (err) {
      console.error("Save Error:", err);
    }
  };

  const handleLogoChange = (event) => {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => saveSettings({ logoUrl: reader.result });
      reader.readAsDataURL(file);
    }
  };

  const toggleDateStatus = async (dateStr) => {
    const updated = { ...managedDates, [dateStr]: !managedDates[dateStr] };
    await saveSettings({ managedDates: updated });
  };

  const addProject = async () => {
    if (newProj.name && user) {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'projects'), {
        ...newProj,
        price: Number(newProj.price) || 0,
        points: Number(newProj.points) || 0
      });
      setNewProj({ name: '', type: 'regular', date: '', timeSlot: '', price: '', points: '' });
    }
  };

  const deleteProject = async (id) => {
    if (!user) return;
    await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'projects', id));
  };

  const togglePayment = async (id, currentStatus) => {
    if (!user) return;
    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'bookings', id), { isPaid: !currentStatus });
  };

  const deleteBooking = async (id) => {
    if (!user) return;
    await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'bookings', id));
  };

  const handleBookingSubmit = async () => {
    if (!user) return;
    const newRecord = {
      name: bookingForm.name,
      phone: bookingForm.phone,
      project: bookingForm.project.name,
      date: bookingForm.project.type === 'special' ? bookingForm.project.date : bookingForm.date,
      time: bookingForm.project.type === 'special' ? bookingForm.project.timeSlot : bookingForm.time,
      price: bookingForm.project.price || 0,
      points: bookingForm.project.points || 0,
      payMethod: bookingForm.project.type === 'special' ? bookingForm.payMethod : 'none',
      lastFive: bookingForm.lastFive || '',
      isPaid: false,
      createdAt: new Date().toISOString()
    };
    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'bookings'), newRecord);
    setStep(4);
  };

  // --- 計算邏輯 ---
  
  const availableDates = Object.entries(managedDates)
    .filter(([dateStr, isOpen]) => {
      if (!isOpen) return false;
      const d = new Date(dateStr);
      const today = new Date();
      return d.getMonth() === today.getMonth() && d.getFullYear() === today.getFullYear();
    })
    .map(([dateStr]) => {
      const d = new Date(dateStr);
      const weekDays = ['週日', '週一', '週二', '週三', '週四', '週五', '週六'];
      return `${dateStr} (${weekDays[d.getDay()]})`;
    })
    .sort();

  const groupedBookings = bookings.reduce((acc, curr) => {
    const slotKey = `${curr.date} ${curr.time}`;
    if (!acc[slotKey]) acc[slotKey] = [];
    acc[slotKey].push(curr);
    return acc;
  }, {});

  if (loading) return (
    <div className="min-h-screen flex flex-col items-center justify-center bg-[#FFF9E6] text-[#B34233]">
      <div className="animate-spin mb-4"><Settings size={32} /></div>
      <p className="font-bold tracking-widest text-sm">正在連接雲端資料庫...</p>
    </div>
  );

  return (
    <div className="min-h-screen bg-[#FFF9E6] font-sans text-[#5D4037] pb-10">
      <button 
        onClick={() => setIsAdmin(!isAdmin)} 
        className="fixed top-4 right-4 z-50 bg-white shadow-lg p-3 rounded-full text-[#B34233] hover:scale-110 transition-transform active:scale-95 border-2 border-[#B34233]/10"
      >
        {isAdmin ? <Users size={20} /> : <Settings size={20} />}
      </button>

      <input type="file" ref={fileInputRef} className="hidden" accept="image/*" onChange={handleLogoChange} />

      <header className="bg-white pt-10 pb-8 flex flex-col items-center shadow-sm border-b-4 border-[#B34233]/10">
        <div className={`w-48 mb-4 ${isAdmin ? 'cursor-pointer' : ''} relative group transition-all duration-500`} onClick={() => isAdmin && fileInputRef.current.click()}>
           <img src={logoUrl} alt="Logo" className="w-full h-auto object-contain rounded-xl" />
           {isAdmin && (
             <div className="absolute inset-0 flex items-center justify-center bg-black/20 opacity-0 group-hover:opacity-100 transition-opacity rounded-xl">
               <Camera className="text-white" />
             </div>
           )}
        </div>
        <h1 className="text-3xl font-bold tracking-[0.3em] text-[#B34233] mb-1 text-center">童 話 森 林</h1>
        <p className="text-[10px] tracking-[0.2em] font-medium opacity-60 uppercase">Handmade ｜ Select ｜ Toys</p>
      </header>

      {isAdmin ? (
        <main className="max-w-6xl mx-auto px-6 py-8 grid grid-cols-1 lg:grid-cols-12 gap-8 animate-in fade-in duration-500">
          <div className="lg:col-span-4 space-y-6">
            <div className="bg-white rounded-3xl p-6 shadow-sm border border-[#B34233]/5">
              <h2 className="text-lg font-bold mb-4 flex items-center gap-2 text-[#B34233]">
                <CalendarIcon size={20} /> 日期排程管理
              </h2>
              <div className="grid grid-cols-4 gap-2 max-h-[350px] overflow-y-auto pr-2">
                {Object.keys(managedDates).sort().map(dateStr => {
                  const d = new Date(dateStr);
                  const isOpen = managedDates[dateStr];
                  return (
                    <button 
                      key={dateStr}
                      onClick={() => toggleDateStatus(dateStr)}
                      className={`flex flex-col items-center py-2 rounded-xl border-2 transition-all ${isOpen ? 'border-[#B34233] bg-[#B34233]/5' : 'border-gray-100 bg-gray-50 opacity-40'}`}
                    >
                      <span className="text-[9px] font-black opacity-40">{d.getMonth()+1}/{d.getDate()}</span>
                      <span className="text-[10px] font-bold">{isOpen ? '開課' : '休'}</span>
                    </button>
                  );
                })}
              </div>
            </div>

            <div className="bg-white rounded-3xl p-6 shadow-sm border border-[#B34233]/5">
              <h2 className="text-lg font-bold mb-6 flex items-center gap-2 text-[#B34233]">
                <LayoutGrid size={20} /> 課程清單
              </h2>
              <div className="bg-[#FFF9E6] p-5 rounded-2xl mb-6">
                <input placeholder="課程名稱" className="w-full mb-3 p-3 rounded-xl border-none shadow-sm text-sm outline-none" value={newProj.name} onChange={e => setNewProj({...newProj, name: e.target.value})} />
                <div className="flex gap-4 mb-4 px-1">
                  <label className="text-xs flex items-center gap-2 cursor-pointer font-bold"><input type="radio" checked={newProj.type === 'regular'} onChange={() => setNewProj({...newProj, type: 'regular'})} /> 常態</label>
                  <label className="text-xs flex items-center gap-2 cursor-pointer font-bold"><input type="radio" checked={newProj.type === 'special'} onChange={() => setNewProj({...newProj, type: 'special'})} /> 特別</label>
                </div>
                {newProj.type === 'special' && (
                  <div className="space-y-3 animate-in slide-in-from-top-2">
                    <div className="grid grid-cols-2 gap-2">
                        <input type="number" placeholder="費用" className="w-full p-3 rounded-xl text-sm border-none shadow-sm" value={newProj.price} onChange={e => setNewProj({...newProj, price: e.target.value})} />
                        <input type="number" placeholder="點數" className="w-full p-3 rounded-xl text-sm border-none shadow-sm" value={newProj.points} onChange={e => setNewProj({...newProj, points: e.target.value})} />
                    </div>
                    <input placeholder="日期 (如 4/5)" className="w-full p-3 rounded-xl text-sm border-none shadow-sm" value={newProj.date} onChange={e => setNewProj({...newProj, date: e.target.value})} />
                    <input placeholder="時間" className="w-full p-3 rounded-xl text-sm border-none shadow-sm" value={newProj.timeSlot} onChange={e => setNewProj({...newProj, timeSlot: e.target.value})} />
                  </div>
                )}
                <button onClick={addProject} className="w-full mt-2 bg-[#B34233] text-white py-3 rounded-xl text-sm font-bold shadow-lg shadow-[#B34233]/20">新增課程</button>
              </div>
              <div className="space-y-2 max-h-[200px] overflow-y-auto pr-1">
                {projects.map(p => (
                  <div key={p.id} className="bg-gray-50 p-3 rounded-2xl flex justify-between items-center text-sm border border-gray-100">
                    <div>
                      <span className="font-bold">{p.name}</span>
                      <p className="text-[10px] opacity-40">{p.type === 'special' ? '特別課' : '常態'}</p>
                    </div>
                    <button onClick={() => deleteProject(p.id)} className="text-gray-200 hover:text-red-500 transition-colors"><Trash2 size={16}/></button>
                  </div>
                ))}
              </div>
            </div>
          </div>

          <div className="lg:col-span-8">
            <div className="bg-white rounded-[2.5rem] p-8 shadow-sm border-2 border-[#B34233]/10 min-h-[600px]">
              <h2 className="text-xl font-bold mb-8 flex items-center gap-2 text-[#B34233]">
                <ListOrdered size={24} /> 預約清單
              </h2>
              <div className="space-y-4">
                {Object.entries(groupedBookings).sort().map(([slot, persons]) => (
                  <div key={slot} className="border border-gray-100 rounded-3xl overflow-hidden shadow-sm">
                    <button 
                      onClick={() => setExpandedSlot(expandedSlot === slot ? null : slot)} 
                      className={`w-full flex items-center justify-between p-5 ${expandedSlot === slot ? 'bg-[#B34233] text-white' : 'bg-white'}`}
                    >
                      <div className="flex items-center gap-3 text-sm font-bold">
                        <Clock size={18} /> {slot}
                        <span className={`text-[10px] px-2 py-0.5 rounded-full ${expandedSlot === slot ? 'bg-white/20 text-white' : 'bg-[#B34233]/10 text-[#B34233]'}`}>
                          {persons.length} 位
                        </span>
                      </div>
                      {expandedSlot === slot ? <ChevronUp size={20} /> : <ChevronDown size={20} />}
                    </button>
                    {expandedSlot === slot && (
                      <div className="bg-[#FFF9E6]/30 p-3 space-y-3">
                        {persons.map(person => (
                          <div key={person.id} className="bg-white p-5 rounded-2xl flex justify-between items-center shadow-sm">
                            <div>
                                <div className="flex items-center gap-2 mb-1">
                                    <span className="font-bold">{person.name}</span>
                                    <span className="text-[10px] bg-[#B34233]/10 text-[#B34233] px-2 py-0.5 rounded font-black">{person.project}</span>
                                </div>
                                <p className="text-[11px] text-gray-400 font-medium">
                                  {person.phone} {person.lastFive ? `｜ 後五碼: ${person.lastFive}` : ''}
                                </p>
                            </div>
                            <div className="flex items-center gap-3">
                                <button 
                                  onClick={() => togglePayment(person.id, person.isPaid)} 
                                  className={`px-4 py-2 rounded-full text-[10px] font-bold transition-all ${person.isPaid ? 'bg-emerald-100 text-emerald-600' : 'bg-gray-100 text-gray-400'}`}
                                >
                                    {person.isPaid ? '已核對' : '確認匯款'}
                                </button>
                                <button onClick={() => deleteBooking(person.id)} className="text-gray-200 hover:text-red-500 transition-colors"><X size={18} /></button>
                            </div>
                          </div>
                        ))}
                      </div>
                    )}
                  </div>
                ))}
              </div>
            </div>
          </div>
        </main>
      ) : (
        <main className="max-w-md mx-auto px-6 py-10">
          {step < 4 && (
            <div className="flex justify-between mb-12 relative px-4">
                <div className="absolute top-1/2 left-0 w-full h-0.5 bg-[#B34233]/10 -translate-y-1/2 z-0"></div>
                {[1, 2, 3].map(num => (
                    <div key={num} className={`relative z-10 w-10 h-10 rounded-full flex items-center justify-center text-sm font-bold transition-all duration-300 ${step >= num ? 'bg-[#B34233] text-white shadow-lg' : 'bg-white border-2 border-[#B34233]/20 text-[#B34233]'}`}>
                        {num}
                    </div>
                ))}
            </div>
          )}

          {step === 1 && (
            <section className="animate-in fade-in duration-500">
              <h2 className="text-xl font-bold mb-8 flex items-center gap-2 text-[#B34233]"><Palette size={22} /> 1. 選擇預約項目</h2>
              <div className="grid grid-cols-2 gap-4">
                {projects.map(p => (
                  <button 
                    key={p.id} 
                    onClick={() => setBookingForm({...bookingForm, project: p})} 
                    className={`p-6 rounded-[2rem] border-2 text-left transition-all relative ${bookingForm.project?.id === p.id ? 'border-[#B34233] bg-[#B34233]/5 shadow-inner' : 'border-white bg-white shadow-sm'}`}
                  >
                    <span className="text-sm font-bold block mb-1">{p.name}</span>
                    <span className="text-[10px] font-black opacity-30 uppercase tracking-tighter">{p.type === 'special' ? '特別課程' : '常態開課'}</span>
                    {bookingForm.project?.id === p.id && <div className="absolute top-3 right-3 text-[#B34233]"><CheckCircle2 size={16} /></div>}
                  </button>
                ))}
              </div>
              <button disabled={!bookingForm.project} onClick={() => setStep(2)} className="w-full mt-12 bg-[#B34233] text-white py-5 rounded-2xl font-bold shadow-xl shadow-[#B34233]/20 disabled:opacity-20">下一步：選擇時間</button>
            </section>
          )}

          {step === 2 && (
            <section className="animate-in slide-in-from-right-8 duration-500">
              <h2 className="text-xl font-bold mb-8 flex items-center gap-2 text-[#B34233]"><CalendarDays size={22} /> 2. 確認時間</h2>
              {bookingForm.project.type === 'special' ? (
                <div className="bg-white p-12 rounded-[3rem] border-2 border-[#B34233] text-center shadow-sm">
                  <p className="text-[10px] font-bold text-[#B34233] mb-4 uppercase tracking-[0.2em] opacity-40">特別課程限定時段</p>
                  <p className="text-4xl font-black text-[#B34233]">{bookingForm.project.date}</p>
                  <div className="h-0.5 w-8 bg-[#B34233]/20 mx-auto my-6"></div>
                  <p className="text-xl font-bold opacity-80">{bookingForm.project.timeSlot}</p>
                </div>
              ) : (
                <div className="space-y-10">
                  <div>
                    <div className="flex justify-between items-end mb-4 px-1">
                      <label className="text-[10px] font-bold opacity-40 uppercase">選擇日期 (目前僅開放本月)</label>
                      <span className="text-[9px] bg-emerald-50 text-emerald-600 px-2 py-0.5 rounded font-black">Open</span>
                    </div>
                    <div className="grid grid-cols-1 gap-2 max-h-[250px] overflow-y-auto pr-1">
                      {availableDates.map(d => (
                        <button key={d} onClick={() => setBookingForm({...bookingForm, date: d})} className={`p-5 rounded-2xl border-2 text-left transition-all ${bookingForm.date === d ? 'border-[#B34233] bg-[#B34233]/5 font-bold' : 'bg-white border-white text-gray-400'}`}>
                          {d}
                        </button>
                      ))}
                      {availableDates.length === 0 && <p className="text-center py-10 text-xs opacity-30">本月目前尚無開放日期</p>}
                    </div>
                  </div>
                  <div>
                    <label className="text-[10px] font-bold mb-4 block opacity-40 px-1 uppercase">選擇時段</label>
                    <div className="grid grid-cols-3 gap-3">
                      {times.map(t => (
                        <button key={t} onClick={() => setBookingForm({...bookingForm, time: t})} className={`py-4 rounded-xl border-2 font-bold text-sm transition-all ${bookingForm.time === t ? 'bg-[#B34233] text-white border-[#B34233]' : 'bg-white border-white text-gray-400'}`}>
                          {t}
                        </button>
                      ))}
                    </div>
                  </div>
                </div>
              )}
              <div className="flex gap-4 mt-12">
                <button onClick={() => setStep(1)} className="flex-1 opacity-40 font-bold text-sm">返回</button>
                <button 
                  onClick={() => setStep(3)} 
                  disabled={bookingForm.project.type === 'regular' && (!bookingForm.date || !bookingForm.time)} 
                  className="flex-[2] bg-[#B34233] text-white py-5 rounded-2xl font-bold shadow-xl shadow-[#B34233]/20 disabled:opacity-20"
                >
                  填寫基本資料
                </button>
              </div>
            </section>
          )}

          {step === 3 && (
            <section className="animate-in slide-in-from-right-8 duration-500 pb-10">
              <h2 className="text-xl font-bold mb-8 flex items-center gap-2 text-[#B34233]"><User size={22} /> 3. 最後確認</h2>
              <div className="space-y-4 mb-8">
                <input type="text" placeholder="學員姓名" className="w-full p-5 rounded-2xl bg-white shadow-sm outline-none focus:ring-2 focus:ring-[#B34233]/10" onChange={(e) => setBookingForm({...bookingForm, name: e.target.value})} />
                <input type="tel" placeholder="聯絡電話" className="w-full p-5 rounded-2xl bg-white shadow-sm outline-none focus:ring-2 focus:ring-[#B34233]/10" onChange={(e) => setBookingForm({...bookingForm, phone: e.target.value})} />
                
                {bookingForm.project.type === 'special' && (
                    <div className="bg-white p-6 rounded-[2.5rem] border-2 border-[#B34233]/10 space-y-5">
                        <p className="text-[10px] font-black text-[#B34233]/50 uppercase text-center tracking-widest">付款資訊</p>
                        <div className="flex gap-2">
                            <button onClick={() => setBookingForm({...bookingForm, payMethod: 'transfer'})} className={`flex-1 py-3 rounded-xl text-xs font-bold transition-all ${bookingForm.payMethod === 'transfer' ? 'bg-[#B34233] text-white' : 'bg-gray-100 text-gray-400'}`}>轉帳</button>
                            <button onClick={() => setBookingForm({...bookingForm, payMethod: 'pointCard'})} className={`flex-1 py-3 rounded-xl text-xs font-bold transition-all ${bookingForm.payMethod === 'pointCard' ? 'bg-[#B34233] text-white' : 'bg-gray-100 text-gray-400'}`}>扣點卷</button>
                        </div>
                        {bookingForm.payMethod === 'transfer' ? (
                            <div className="animate-in fade-in duration-300">
                                <div className="text-[11px] font-bold text-[#B34233]/70 mb-3 space-y-1">
                                    <p>銀行：({bankInfo.code}) 中信</p>
                                    <p>帳號：{bankInfo.account}</p>
                                    <p className="text-sm font-black text-[#B34233]">金額：${bookingForm.project.price}</p>
                                </div>
                                <input maxLength="5" placeholder="請輸入匯款後五碼" className="w-full p-4 rounded-xl bg-gray-50 text-sm shadow-inner outline-none" onChange={(e) => setBookingForm({...bookingForm, lastFive: e.target.value})} />
                            </div>
                        ) : (
                            <div className="bg-orange-50 p-4 rounded-xl text-center border border-orange-100">
                                <p className="text-xs font-bold text-orange-600">當天現場收取 {bookingForm.project.points} 張點數卷</p>
                            </div>
                        )}
                    </div>
                )}
              </div>

              <div className="flex gap-4">
                <button onClick={() => setStep(2)} className="flex-1 opacity-40 font-bold text-sm">返回</button>
                <button 
                  onClick={handleBookingSubmit} 
                  disabled={!bookingForm.name || !bookingForm.phone || (bookingForm.project.type === 'special' && bookingForm.payMethod === 'transfer' && !bookingForm.lastFive)} 
                  className="flex-[2] bg-[#B34233] text-white py-5 rounded-2xl font-bold shadow-xl shadow-[#B34233]/20 active:scale-95 transition-transform"
                >
                  確認預約
                </button>
              </div>
            </section>
          )}

          {step === 4 && (
            <section className="text-center py-20 animate-in zoom-in duration-500">
              <div className="w-24 h-24 bg-emerald-50 text-emerald-500 rounded-full flex items-center justify-center mx-auto mb-8"><CheckCircle2 size={48} /></div>
              <h2 className="text-3xl font-black text-[#B34233] mb-4">預約成功！</h2>
              <p className="text-sm opacity-50 mb-12">我們已收到預約，老師核對後會與您聯繫。</p>
              <button onClick={() => setStep(1)} className="text-[#B34233] underline font-black tracking-widest text-xs uppercase">返回首頁</button>
            </section>
          )}
        </main>
      )}

      <footer className="text-center mt-auto py-10 opacity-20">
        <p className="text-[10px] tracking-[0.4em] flex items-center justify-center gap-2 uppercase font-black">
           <Heart size={10} className="fill-[#B34233]" /> 童話森林手作工作室
        </p>
      </footer>
    </div>
  );
};

export default App;