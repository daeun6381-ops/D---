import React, { useState, useEffect, useRef } from 'react';
import { 
  ChevronLeft, 
  ChevronRight, 
  Plus, 
  Trash2, 
  Settings,
  Clock, 
  Play, 
  Pause, 
  RotateCcw
} from 'lucide-react';

// --- Firebase Imports ---
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, collection, query } from 'firebase/firestore';

// ==========================================================
// [설정 완료] FIREBASE CONFIGURATION
// ==========================================================
const firebaseConfig = {
  apiKey: "AIzaSyBv14g9crV8vGbobK5cdVxwqlrr0EiM0fA",
  authDomain: "bokk-55eae.firebaseapp.com",
  projectId: "bokk-55eae",
  storageBucket: "bokk-55eae.firebasestorage.app",
  messagingSenderId: "757990079253",
  appId: "1:757990079253:web:f3dbf171b2137f436bc71c",
  measurementId: "G-2PMDNT4LM2"
};

// Initialize Firebase services
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const APP_ID = typeof __app_id !== 'undefined' ? __app_id : 'planner-app-v1';

const App = () => {
  // --- State Management ---
  const [user, setUser] = useState(null);
  const [view, setView] = useState('daily');
  const [records, setRecords] = useState({});
  const [loading, setLoading] = useState(true);
  const [dDayTarget, setDDayTarget] = useState('');

  const getTodayStr = () => {
    const d = new Date();
    return `${d.getFullYear()}${String(d.getMonth() + 1).padStart(2, '0')}${String(d.getDate()).padStart(2, '0')}`;
  };

  const [currentDate, setCurrentDate] = useState(getTodayStr());

  // --- Stopwatch State ---
  const [isTimerRunning, setIsTimerRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const timerRef = useRef(null);

  const emptyRecord = {
    date: currentDate,
    resolution: "",
    todos: [],
    totalTime: 0 
  };

  const currentRecord = records[currentDate] || { ...emptyRecord, date: currentDate };

  // --- Firebase Auth & Firestore Sync Logic ---
  useEffect(() => {
    let unsubscribeDDay = null;
    let unsubscribeRecords = null;

    const performSignIn = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          try {
            await signInWithCustomToken(auth, __initial_auth_token);
          } catch (tokenErr) {
            console.warn("Custom token failed, falling back to anonymous auth:", tokenErr.message);
            await signInAnonymously(auth);
          }
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth Error:", err);
        setLoading(false);
      }
    };

    performSignIn();

    const unsubscribeAuth = onAuthStateChanged(auth, (currentUser) => {
      if (currentUser) {
        setUser(currentUser);
        
        if (unsubscribeDDay) unsubscribeDDay();
        if (unsubscribeRecords) unsubscribeRecords();

        const dDayDocRef = doc(db, 'artifacts', APP_ID, 'users', currentUser.uid, 'settings', 'dday');
        unsubscribeDDay = onSnapshot(dDayDocRef, (docSnap) => {
          if (docSnap.exists()) setDDayTarget(docSnap.data().target || '');
        }, (err) => {
          console.error("D-Day Sync Error:", err);
        });

        const recordsColRef = collection(db, 'artifacts', APP_ID, 'users', currentUser.uid, 'records');
        unsubscribeRecords = onSnapshot(recordsColRef, (querySnapshot) => {
          const data = {};
          querySnapshot.forEach((doc) => {
            data[doc.id] = doc.data();
          });
          setRecords(data);
          setLoading(false);
        }, (err) => {
          console.error("Records Sync Error:", err);
          setLoading(false);
        });
      } else {
        setUser(null);
      }
    });

    return () => {
      if (unsubscribeAuth) unsubscribeAuth();
      if (unsubscribeDDay) unsubscribeDDay();
      if (unsubscribeRecords) unsubscribeRecords();
    };
  }, []);

  // --- 날짜 변경 시 타이머 동기화 ---
  useEffect(() => {
    setSeconds(currentRecord.totalTime || 0);
    setIsTimerRunning(false);
    if (timerRef.current) clearInterval(timerRef.current);
  }, [currentDate, currentRecord.totalTime]);

  // --- Cloud Save Handler ---
  const saveRecordToCloud = async (updates) => {
    if (!user) return;
    const recordRef = doc(db, 'artifacts', APP_ID, 'users', user.uid, 'records', currentDate);
    try {
      await setDoc(recordRef, { ...currentRecord, ...updates }, { merge: true });
    } catch (e) {
      console.error("Cloud Save Error:", e);
    }
  };

  const handleDDayChange = async (val) => {
    setDDayTarget(val);
    if (!user) return;
    const dDayDocRef = doc(db, 'artifacts', APP_ID, 'users', user.uid, 'settings', 'dday');
    await setDoc(dDayDocRef, { target: val });
  };

  // --- Utilities ---
  const formatDisplayDate = (dateStr) => {
    const year = dateStr.substring(0, 4);
    const month = dateStr.substring(4, 6);
    const day = dateStr.substring(6, 8);
    const d = new Date(`${year}-${month}-${day}`);
    const dayOfWeek = ["일", "월", "화", "수", "목", "금", "토"][d.getDay()];
    return `${year}/${month}/${day}/(${dayOfWeek})`;
  };

  const formatTimeMain = (totalSeconds) => {
    const mins = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${String(mins).padStart(2, '0')}:${String(secs).padStart(2, '0')}`;
  };

  const formatTimeSub = (totalSeconds) => {
    const hrs = Math.floor(totalSeconds / 3600);
    return `${hrs}H`;
  };

  // --- Timer Logic ---
  const toggleTimer = () => {
    if (isTimerRunning) {
      clearInterval(timerRef.current);
      saveRecordToCloud({ totalTime: seconds });
    } else {
      timerRef.current = setInterval(() => {
        setSeconds(prev => prev + 1);
      }, 1000);
    }
    setIsTimerRunning(!isTimerRunning);
  };

  const resetTimer = () => {
    if (timerRef.current) clearInterval(timerRef.current);
    setIsTimerRunning(false);
    setSeconds(0);
    saveRecordToCloud({ totalTime: 0 });
  };

  // --- Todo Handlers ---
  const addTodo = () => {
    const newTodo = { id: Date.now(), text: "", completed: false };
    saveRecordToCloud({ todos: [...currentRecord.todos, newTodo] });
  };

  const updateTodo = (id, text) => {
    const newTodos = currentRecord.todos.map(t => t.id === id ? { ...t, text } : t);
    saveRecordToCloud({ todos: newTodos });
  };

  const toggleTodo = (id) => {
    const newTodos = currentRecord.todos.map(t => t.id === id ? { ...t, completed: !t.completed } : t);
    saveRecordToCloud({ todos: newTodos });
  };

  const removeTodo = (id) => {
    const newTodos = currentRecord.todos.filter(t => t.id !== id);
    saveRecordToCloud({ todos: newTodos });
  };

  const calculateDDay = () => {
    if (!dDayTarget) return "D-?";
    const target = new Date(dDayTarget);
    const today = new Date();
    today.setHours(0,0,0,0);
    const diffDays = Math.ceil((target - today) / (1000 * 60 * 60 * 24));
    return diffDays === 0 ? "D-Day" : diffDays > 0 ? `D-${diffDays}` : `D+${Math.abs(diffDays)}`;
  };

  const changeDate = (offset) => {
    if (isTimerRunning) {
      clearInterval(timerRef.current);
      setIsTimerRunning(false);
      saveRecordToCloud({ totalTime: seconds });
    }
    const d = new Date(currentDate.replace(/(\d{4})(\d{2})(\d{2})/, '$1-$2-$3'));
    d.setDate(d.getDate() + offset);
    const newDateStr = `${d.getFullYear()}${String(d.getMonth() + 1).padStart(2, '0')}${String(d.getDate()).padStart(2, '0')}`;
    setCurrentDate(newDateStr);
    setView('daily');
  };

  const theme = {
    primary: '#BAE6FD',
    secondary: '#E0F2FE',
    point: '#0284C7',
    sidebar: '#0C4A6E',
    timerBg: '#0C4A6E',
  };

  const timetable = [
    ["자율", "문권", "연구", "문박", "B"],
    ["E", "A", "D", "C", "영이"],
    ["A", "B", "영이", "스최", "대손"],
    ["스최", "E", "대손", "대손", "E"],
    ["영유", "대손", "문박", "D", "C"],
    ["D", "영유", "B", "A", "창체"],
    ["문권", "C", "창체", "창체", "창체"]
  ];
  const days = ["월", "화", "수", "목", "금"];

  if (loading) {
    return (
      <div className="h-screen bg-blue-50 flex flex-col items-center justify-center font-bold text-blue-900 gap-4">
        <div className="w-12 h-12 border-4 border-blue-200 border-t-blue-600 rounded-full animate-spin"></div>
        <p>로그인 및 데이터 연동 중...</p>
      </div>
    );
  }

  return (
    <div className="h-screen bg-[#F0F9FF] flex items-center justify-center p-4 overflow-hidden">
      <div className="w-full max-w-[1100px] h-full max-h-[850px] bg-white shadow-2xl flex relative border-l-[30px] overflow-hidden rounded-sm" style={{ borderColor: theme.sidebar }}>
        
       <div className="absolute left-0 top-0 bottom-0 w-8 flex flex-col justify-around py-6 z-10 pointer-events-none">
          {[...Array(14)].map((_, i) => (
            <div key={i} className="w-10 h-3 bg-gradient-to-r from-blue-100 to-white/40 rounded-full -ml-5 shadow-sm border border-white/20" />
          ))}
        </div>

        <div className="flex-1 flex flex-col ml-6 overflow-hidden">
          <div className="px-6 py-2.5 flex justify-between items-center border-b border-blue-50 shrink-0 bg-white">
            <div className="flex items-center gap-6">
              <div className="flex items-center gap-1.5">
                <button onClick={() => changeDate(-1)} className="p-1 hover:bg-blue-50 rounded-full transition-colors"><ChevronLeft className="w-5 h-5 text-blue-300" /></button>
                <h1 className="text-xl font-black tracking-tight text-gray-900 min-w-[140px] text-center">{formatDisplayDate(currentDate)}</h1>
                <button onClick={() => changeDate(1)} className="p-1 hover:bg-blue-50 rounded-full transition-colors"><ChevronRight className="w-5 h-5 text-blue-300" /></button>
              </div>
              
              <div className="flex items-center gap-2">
                <div className="flex items-center gap-2 px-3 py-1 rounded-full border shadow-sm" style={{ backgroundColor: theme.secondary, borderColor: theme.primary }}>
                  <div className="text-lg font-black leading-none tracking-tighter" style={{ color: theme.point }}>{calculateDDay()}</div>
                  <div className="h-2.5 w-[1px] bg-blue-200 mx-0.5" />
                  <input type="date" value={dDayTarget} onChange={(e) => handleDDayChange(e.target.value)} className="text-[9px] bg-transparent border-none p-0 focus:ring-0 outline-none font-bold text-blue-600 cursor-pointer w-24" />
                </div>
                <div className="flex gap-1 p-1 bg-blue-50/50 rounded-full border border-blue-100">
                  <button onClick={() => setView('daily')} className={`px-4 py-1 text-[10px] font-black rounded-full transition-all ${view === 'daily' ? 'bg-white shadow-md' : 'text-blue-300'}`} style={view === 'daily' ? { color: theme.point } : {}}>PLANNER</button>
                  <button onClick={() => setView('archive')} className={`px-4 py-1 text-[10px] font-black rounded-full transition-all ${view === 'archive' ? 'bg-white shadow-md' : 'text-blue-300'}`} style={view === 'archive' ? { color: theme.point } : {}}>ARCHIVE</button>
                </div>
              </div>
            </div>
            <div className="flex items-center gap-3">
              <span className="text-[8px] text-blue-200 font-mono">UID: {user?.uid}</span>
              <Settings className="w-4 h-4 text-blue-200 cursor-pointer hover:rotate-45" />
            </div>
          </div>

          {view === 'daily' ? (
            <div className="flex-1 flex overflow-hidden">
              <div className="flex-[1.8] border-r border-blue-50 p-5 overflow-hidden flex flex-col">
                <div className="mb-4 p-3 rounded-xl border-2 flex items-center gap-3 shrink-0" style={{ backgroundColor: '#F0F9FF', borderColor: theme.primary }}>
                   <div className="w-1.5 h-6 rounded-full" style={{ backgroundColor: theme.point }} />
                   <textarea placeholder="오늘의 다짐 한 문장..." value={currentRecord.resolution} onChange={(e) => saveRecordToCloud({ resolution: e.target.value })} className="flex-1 bg-transparent text-[16px] font-black border-none focus:ring-0 resize-none h-6 leading-tight placeholder:text-blue-200 checklist-font text-gray-800" />
                </div>
                <div className="flex-1 overflow-y-auto scrollbar-hide">
                  <div className="flex justify-between items-center mb-4 sticky top-0 bg-white z-10 pb-2">
                    <span className="text-[11px] font-black text-white tracking-widest px-3 py-1 rounded-full shadow-md uppercase" style={{ backgroundColor: theme.point }}>Tasks</span>
                    <button onClick={addTodo} className="w-9 h-9 text-white rounded-full flex items-center justify-center shadow-lg hover:scale-110 active:scale-95" style={{ backgroundColor: theme.sidebar }}><Plus className="w-7 h-7" /></button>
                  </div>
                  <div className="space-y-2 px-1 pb-10">
                    {currentRecord.todos.map((todo) => (
                      <div key={todo.id} className="flex items-center gap-4 group">
                        <button onClick={() => toggleTodo(todo.id)} className="shrink-0 relative">
                          <div className={`w-8 h-8 rounded-lg border-2 transition-all ${todo.completed ? 'scale-90' : 'border-blue-100 bg-blue-50/30'}`} style={todo.completed ? { borderColor: theme.primary, backgroundColor: theme.primary } : {}} />
                          {todo.completed && <svg className="absolute inset-0 m-auto w-5 h-5 text-blue-800" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="5" d="M5 13l4 4L19 7" /></svg>}
                        </button>
                        <input type="text" value={todo.text} onChange={(e) => updateTodo(todo.id, e.target.value)} placeholder="할 일을 입력하세요..." className={`flex-1 bg-transparent border-none focus:ring-0 text-[18px] p-1 checklist-font ${todo.completed ? 'text-blue-200 line-through italic' : 'text-gray-800'}`} />
                        <button onClick={() => removeTodo(todo.id)} className="opacity-0 group-hover:opacity-100 p-1.5 text-gray-200 hover:text-red-400"><Trash2 className="w-4 h-4" /></button>
                      </div>
                    ))}
                  </div>
                </div>
              </div>
              <div className="flex-[1] p-5 flex flex-col gap-4">
                <div className="flex-1 flex flex-col min-h-0">
                  <div className="flex items-center gap-2 mb-3"><Clock className="w-3.5 h-3.5" style={{ color: theme.point }} /><h3 className="text-[10px] font-black text-blue-800 tracking-[0.2em] uppercase">Weekly Timetable</h3></div>
                  <div className="flex-1 bg-white border-2 border-blue-50 rounded-xl overflow-hidden flex flex-col shadow-sm">
                    <div className="grid grid-cols-5 shrink-0" style={{ backgroundColor: theme.sidebar }}>
                      {days.map(day => <div key={day} className="py-1 text-center text-[9px] font-black text-blue-100 border-r border-white/10 uppercase last:border-r-0">{day}</div>)}
                    </div>
                    <div className="flex-1 grid grid-rows-7 overflow-hidden">
                      {timetable.map((row, r) => <div key={r} className="grid grid-cols-5 border-b border-blue-50 last:border-b-0">
                        {row.map((sub, c) => (
                          <div key={c} className="p-0.5 border-r border-blue-50 flex items-center justify-center text-center last:border-r-0 hover:bg-blue-50 transition-colors">
                            <span className="font-bold text-gray-700 text-[13px] leading-tight line-clamp-2">{sub}</span>
                          </div>
                        ))}
                      </div>)}
                    </div>
                  </div>
                </div>
                <div className="shrink-0">
                  <div className="rounded-2xl p-2 px-3 flex items-center justify-between shadow-xl border border-white/20" style={{ backgroundColor: theme.timerBg }}>
                    <button onClick={toggleTimer} className={`w-11 h-11 rounded-full flex items-center justify-center transition-all ${isTimerRunning ? 'text-blue-900 shadow-[0_0_20px_rgba(186,230,253,0.6)]' : 'bg-blue-900/40 hover:bg-blue-800'}`} style={isTimerRunning ? { backgroundColor: theme.primary } : { color: theme.primary }}>
                      {isTimerRunning ? <Pause className="w-5 h-5 fill-current" /> : <Play className="w-5 h-5 fill-current ml-0.5" />}
                    </button>
                    <div className="flex items-baseline gap-1">
                      <span className="text-2xl font-normal tabular-nums digital-font" style={{ color: theme.primary }}>{formatTimeMain(seconds)}</span>
                      <span className="text-[10px] font-bold text-blue-300/80 digital-font">{formatTimeSub(seconds)}</span>
                    </div>
                    <button onClick={resetTimer} className="w-11 h-11 bg-blue-900/40 rounded-full flex items-center justify-center text-blue-400 hover:text-blue-100"><RotateCcw className="w-5 h-5" /></button>
                  </div>
                </div>
              </div>
            </div>
          ) : (
            <div className="flex-1 p-8 overflow-y-auto bg-white scrollbar-hide">
              <h2 className="text-2xl font-black text-gray-900 italic mb-8 uppercase tracking-tighter">My History</h2>
              <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-4">
                {Object.keys(records).sort((a,b) => b.localeCompare(a)).map(date => (
                  <div key={date} className="p-4 rounded-[32px] border-2 border-gray-50 bg-white shadow-md hover:scale-105 transition-all">
                    <div className="text-lg font-black text-gray-900 tracking-tighter">{formatDisplayDate(date)}</div>
                    <div className="text-[10px] text-blue-400 truncate mb-3 italic">"{records[date].resolution || '내용 없음'}"</div>
                    <div className="text-[10px] font-black pt-3 border-t border-blue-50" style={{ color: theme.point }}>{formatTimeSub(records[date].totalTime || 0)} {formatTimeMain(records[date].totalTime || 0)}</div>
                  </div>
                ))}
              </div>
            </div>
          )}
        </div>
      </div>
      <style dangerouslySetInnerHTML={{ __html: `
        @import url('https://fonts.googleapis.com/css2?family=Gowun+Dodum:wght@400;700&family=Orbitron:wght@400;700&display=swap');
        body { font-family: 'Gowun Dodum', sans-serif; background-color: #F0F9FF; margin: 0; overflow: hidden; }
        .digital-font { font-family: 'Orbitron', sans-serif; }
        .checklist-font { font-family: 'Gulim', sans-serif; font-weight: 900; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .line-clamp-2 { display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
      `}} />
    </div>
  );
};

export default App;
