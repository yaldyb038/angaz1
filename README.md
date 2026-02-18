import React, { useState, useEffect, useMemo } from 'react';
import { BookOpen, Headphones, CheckCircle, XCircle, Users, Calendar, BarChart, Settings, Plus, Trash2, Save, LogOut, Loader2, Lock, CalendarClock, ToggleLeft, ToggleRight, AlertTriangle, Trophy, Medal } from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { getFirestore, collection, addDoc, deleteDoc, doc, onSnapshot, updateDoc, setDoc } from 'firebase/firestore';

// ---------------------------------------------------------
// إعدادات Firebase (تم التعديل لتعمل تلقائياً)
// ---------------------------------------------------------

let firebaseConfig;
let appId;

// 1. محاولة استخدام إعدادات البيئة التجريبية (لكي يعمل الموقع الآن أمامك)
if (typeof __firebase_config !== 'undefined') {
  firebaseConfig = JSON.parse(__firebase_config);
  appId = typeof __app_id !== 'undefined' ? __app_id : 'ramadan-tracker-v1';
} else {
  // 2. إعدادات النشر (Production)
  // عندما تنشر الموقع على Vercel أو غيره، سيستخدم هذه البيانات.
  // عليك تعبئة هذه البيانات من لوحة تحكم Firebase الخاصة بك.
  firebaseConfig = {
    apiKey: "ضع_API_KEY_الخاص_بك_هنا",
    authDomain: "YOUR_PROJECT.firebaseapp.com",
    projectId: "YOUR_PROJECT_ID",
    storageBucket: "YOUR_PROJECT.appspot.com",
    messagingSenderId: "SENDER_ID",
    appId: "APP_ID"
  };
  appId = 'ramadan-tracker-production';
}

// تهيئة التطبيق
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

// --- المكونات البسيطة ---
const Card = ({ children, className = "" }) => (
  <div className={`bg-white rounded-xl shadow-lg border border-emerald-100 overflow-hidden ${className}`}>
    {children}
  </div>
);

const Button = ({ children, onClick, variant = "primary", className = "", disabled = false, type = "button" }) => {
  const baseStyle = "px-4 py-2 rounded-lg font-medium transition-all duration-200 flex items-center justify-center gap-2";
  const variants = {
    primary: "bg-emerald-600 text-white hover:bg-emerald-700 active:scale-95 shadow-md hover:shadow-emerald-200",
    secondary: "bg-amber-500 text-white hover:bg-amber-600 active:scale-95 shadow-md hover:shadow-amber-200",
    outline: "border-2 border-emerald-600 text-emerald-700 hover:bg-emerald-50",
    danger: "bg-red-500 text-white hover:bg-red-600 shadow-sm",
    ghost: "bg-gray-100 text-gray-600 hover:bg-gray-200"
  };
  return (
    <button 
      type={type}
      onClick={onClick} 
      disabled={disabled}
      className={`${baseStyle} ${variants[variant]} ${className} ${disabled ? 'opacity-50 cursor-not-allowed' : ''}`}
    >
      {children}
    </button>
  );
};

// مكون عرض الأعمدة
const StatusColumn = ({ title, records, icon, headerColor, onDeleteRequest }) => (
  <div className="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden flex flex-col h-full">
    <div className={`p-3 border-b flex justify-between items-center ${headerColor}`}>
      <div className="flex items-center gap-2 font-bold text-sm">
        {icon}
        {title}
      </div>
      <span className="bg-white/50 px-2 py-0.5 rounded-full text-xs font-bold">
        {records.length}
      </span>
    </div>
    <div className="p-2 bg-gray-50/50 flex-1 min-h-[100px]">
      {records.length === 0 ? (
        <div className="h-full flex items-center justify-center text-gray-400 text-xs italic p-4">
          لا يوجد سجلات
        </div>
      ) : (
        <ul className="space-y-2">
          {records.map(r => (
            <li key={r.id} className="relative bg-white p-2 rounded shadow-sm border border-gray-100 flex justify-between items-center group">
              <span className="text-sm font-medium text-gray-700 truncate max-w-[120px]" title={r.student}>
                {r.student}
              </span>
              <button 
                type="button"
                onClick={(e) => {
                  e.stopPropagation();
                  onDeleteRequest(r.id); 
                }} 
                className="relative z-10 flex items-center justify-center w-8 h-8 text-red-500 bg-red-50 hover:bg-red-100 hover:text-red-700 rounded-full transition-all cursor-pointer active:scale-90 border border-red-100 shadow-sm"
                title="حذف السجل"
              >
                <XCircle size={18} className="pointer-events-none"/>
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  </div>
);

// مكون لوحة المتصدرين
const Leaderboard = ({ records }) => {
  const leaderboardData = useMemo(() => {
    const counts = {};
    records.forEach(r => {
      if (r.status === 'both') {
        counts[r.student] = (counts[r.student] || 0) + 1;
      }
    });
    return Object.entries(counts)
      .map(([name, count]) => ({ name, count }))
      .sort((a, b) => b.count - a.count);
  }, [records]);

  return (
    <Card className="h-full border-amber-200">
      <div className="bg-gradient-to-br from-amber-100 to-orange-50 p-4 border-b border-amber-200">
        <h3 className="text-lg font-bold text-amber-800 flex items-center gap-2">
          <Trophy className="w-6 h-6 text-amber-500 fill-amber-500 animate-pulse"/>
          فرسان رمضان
        </h3>
        <p className="text-xs text-amber-700 mt-1 opacity-80">الأكثر إنجازاً للقراءة والسماع معاً</p>
      </div>
      
      <div className="p-0 overflow-y-auto max-h-[500px] scrollbar-thin">
        {leaderboardData.length === 0 ? (
          <div className="p-8 text-center text-gray-400 text-sm flex flex-col items-center">
            <Medal className="w-12 h-12 mb-2 opacity-20"/>
            لا يوجد منافسين حتى الآن..
            <br/>كن أول المبادرين!
          </div>
        ) : (
          <div className="divide-y divide-amber-100">
            {leaderboardData.map((item, index) => {
              let rankIcon = null;
              let bgClass = "bg-white hover:bg-amber-50/50";
              let textClass = "text-gray-700";
              
              if (index === 0) {
                rankIcon = <div className="w-8 h-8 rounded-full bg-yellow-100 text-yellow-600 flex items-center justify-center font-bold border-2 border-yellow-200 shadow-sm">1</div>;
                bgClass = "bg-gradient-to-r from-yellow-50 to-white";
                textClass = "text-yellow-800 font-bold";
              } else if (index === 1) {
                rankIcon = <div className="w-8 h-8 rounded-full bg-slate-100 text-slate-500 flex items-center justify-center font-bold border-2 border-slate-200">2</div>;
                textClass = "text-slate-700 font-semibold";
              } else if (index === 2) {
                rankIcon = <div className="w-8 h-8 rounded-full bg-orange-100 text-orange-600 flex items-center justify-center font-bold border-2 border-orange-200">3</div>;
                textClass = "text-orange-800 font-semibold";
              } else {
                rankIcon = <div className="w-8 h-8 rounded-full bg-gray-50 text-gray-400 flex items-center justify-center font-bold text-sm">{index + 1}</div>;
              }

              return (
                <div key={item.name} className={`flex items-center justify-between p-4 transition-colors ${bgClass}`}>
                  <div className="flex items-center gap-3">
                    {rankIcon}
                    <span className={`text-base ${textClass}`}>{item.name}</span>
                  </div>
                  <div className="flex items-center gap-1 bg-white px-2 py-1 rounded-md border border-amber-100 shadow-sm">
                    <span className="font-bold text-emerald-600">{item.count}</span>
                    <span className="text-[10px] text-gray-400">إنجاز</span>
                  </div>
                </div>
              );
            })}
          </div>
        )}
      </div>
    </Card>
  );
};

export default function RamadanTracker() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  const [students, setStudents] = useState([]);
  const [records, setRecords] = useState([]);
  
  const fullDays = Array.from({ length: 30 }, (_, i) => i + 1);
  const [activeDays, setActiveDays] = useState(fullDays);

  const [isAdmin, setIsAdmin] = useState(false);
  const [showAdminLogin, setShowAdminLogin] = useState(false);
  const [adminPasswordInput, setAdminPasswordInput] = useState('');
  const [loginError, setLoginError] = useState('');
  
  const [confirmModal, setConfirmModal] = useState({ show: false, message: '', onConfirm: null });
  const [viewDay, setViewDay] = useState('1'); 

  const ADMIN_PASSWORD = "88778"; 

  const [activeTab, setActiveTab] = useState('form');
  const [message, setMessage] = useState(null);
  const [submitting, setSubmitting] = useState(false);

  const [selectedStudent, setSelectedStudent] = useState('');
  const [selectedDay, setSelectedDay] = useState('');
  const [status, setStatus] = useState('');
  const [newStudentName, setNewStudentName] = useState('');

  // --- المصادقة ---
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("Auth Error:", error);
      }
    };
    initAuth();
    
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      if (currentUser) setLoading(false);
    });
    return () => unsubscribe();
  }, []);

  // --- جلب البيانات ---
  useEffect(() => {
    if (!user) return;

    // استخدام المجموعة المباشرة بدلاً من artifacts للإنتاج الفعلي ليكون المسار أبسط
    // في بيئة المعاينة نستخدم المسار الكامل، وفي الإنتاج نستخدم المسار القصير إذا تم ضبطه
    const collectionPrefix = typeof __app_id !== 'undefined' 
      ? `artifacts/${__app_id}/public/data/` 
      : '';

    const studentsRef = collection(db, `${collectionPrefix}students_list`);
    const unsubStudents = onSnapshot(studentsRef, (snapshot) => {
      const loadedStudents = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setStudents(loadedStudents.sort((a, b) => a.name.localeCompare(b.name)));
    }, (error) => console.error("Students Sync Error:", error));

    const recordsRef = collection(db, `${collectionPrefix}achievement_records`);
    const unsubRecords = onSnapshot(recordsRef, (snapshot) => {
      const loadedRecords = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setRecords(loadedRecords);
    }, (error) => console.error("Records Sync Error:", error));

    const settingsRef = doc(db, `${collectionPrefix}settings`, 'config');
    const unsubSettings = onSnapshot(settingsRef, (docSnap) => {
      if (docSnap.exists()) {
        const data = docSnap.data();
        if (data && Array.isArray(data.activeDays)) {
           setActiveDays(data.activeDays);
        }
      } else {
        setDoc(settingsRef, { activeDays: fullDays });
      }
    }, (error) => console.error("Settings Sync Error:", error));

    return () => {
      unsubStudents();
      unsubRecords();
      unsubSettings();
    };
  }, [user]);

  // --- العمليات ---
  const handleAdminAccess = () => {
    if (isAdmin) { setIsAdmin(false); setActiveTab('form'); } else { setShowAdminLogin(true); }
  };

  const verifyAdminPassword = (e) => {
    e.preventDefault();
    if (adminPasswordInput === ADMIN_PASSWORD) {
      setIsAdmin(true); setShowAdminLogin(false); setAdminPasswordInput(''); setLoginError('');
    } else {
      setLoginError('كلمة المرور غير صحيحة، حاول مرة أخرى.');
    }
  };

  const handleAddStudent = async () => {
    if (!newStudentName.trim() || !user) return;
    if (students.some(s => s.name === newStudentName.trim())) {
      setMessage({ type: 'error', text: 'هذا الاسم موجود مسبقاً' });
      return;
    }
    
    const collectionPrefix = typeof __app_id !== 'undefined' ? `artifacts/${__app_id}/public/data/` : '';

    try {
      await addDoc(collection(db, `${collectionPrefix}students_list`), {
        name: newStudentName.trim(),
        createdAt: new Date().toISOString()
      });
      setNewStudentName('');
      setMessage({ type: 'success', text: 'تم إضافة الطالب بنجاح' });
      setTimeout(() => setMessage(null), 3000);
    } catch (e) {
      console.error(e);
      setMessage({ type: 'error', text: 'حدث خطأ أثناء الإضافة (تأكد من إعدادات الفايربيس)' });
    }
  };

  const requestDeleteStudent = (id, name) => {
    setConfirmModal({
      show: true,
      message: `هل أنت متأكد من حذف الطالب "${name}"؟\nسيتم حذف جميع سجلاته السابقة أيضاً ولن يمكن استعادتها.`,
      onConfirm: () => performDeleteStudent(id, name)
    });
  };

  const performDeleteStudent = async (id, name) => {
    try {
      setConfirmModal({ ...confirmModal, show: false });
      const collectionPrefix = typeof __app_id !== 'undefined' ? `artifacts/${__app_id}/public/data/` : '';
      
      const studentRecords = records.filter(r => r.student === name);
      const deletePromises = studentRecords.map(r => 
        deleteDoc(doc(db, `${collectionPrefix}achievement_records`, r.id))
      );
      await Promise.all(deletePromises);
      await deleteDoc(doc(db, `${collectionPrefix}students_list`, id));
      setMessage({ type: 'success', text: `تم حذف الطالب ${name} وجميع سجلاته` });
      setTimeout(() => setMessage(null), 3000);
    } catch (e) { 
      console.error(e); 
      alert("حدث خطأ أثناء الحذف");
    }
  };

  const requestDeleteRecord = (id) => {
    setConfirmModal({
      show: true,
      message: "هل تريد حذف سجل الإنجاز هذا؟",
      onConfirm: () => performDeleteRecord(id)
    });
  };

  const performDeleteRecord = async (id) => {
    try {
      setConfirmModal({ ...confirmModal, show: false });
      const collectionPrefix = typeof __app_id !== 'undefined' ? `artifacts/${__app_id}/public/data/` : '';
      await deleteDoc(doc(db, `${collectionPrefix}achievement_records`, id));
    } catch (e) { 
      console.error(e);
      alert("حدث خطأ في الحذف");
    }
  };

  const toggleDayAvailability = async (day) => {
    const isCurrentlyActive = activeDays.includes(day);
    let newActiveDays;
    if (isCurrentlyActive) {
      newActiveDays = activeDays.filter(d => d !== day);
    } else {
      newActiveDays = [...activeDays, day].sort((a, b) => a - b);
    }
    const collectionPrefix = typeof __app_id !== 'undefined' ? `artifacts/${__app_id}/public/data/` : '';
    try {
      await setDoc(doc(db, `${collectionPrefix}settings`, 'config'), { activeDays: newActiveDays });
    } catch (e) {
      console.error(e);
      alert("حدث خطأ أثناء تحديث الإعدادات");
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!selectedStudent || !selectedDay || !status || !user) {
      setMessage({ type: 'error', text: 'الرجاء تعبئة جميع الخانات' });
      return;
    }
    setSubmitting(true);
    const collectionPrefix = typeof __app_id !== 'undefined' ? `artifacts/${__app_id}/public/data/` : '';

    try {
      const existingRecord = records.find(r => r.student === selectedStudent && r.day === selectedDay);
      if (existingRecord) {
        await updateDoc(doc(db, `${collectionPrefix}achievement_records`, existingRecord.id), {
           status: status, timestamp: new Date().toISOString(), updatedBy: user.uid
        });
      } else {
        await addDoc(collection(db, `${collectionPrefix}achievement_records`), {
          student: selectedStudent, day: selectedDay, status: status, timestamp: new Date().toISOString(), createdBy: user.uid
        });
      }
      setMessage({ type: 'success', text: 'تم تسجيل الإنجاز بنجاح! تقبل الله طاعتكم' });
      setStatus('');
      setTimeout(() => setMessage(null), 3000);
    } catch (error) {
      console.error(error);
      setMessage({ type: 'error', text: 'حدث خطأ في الاتصال، حاول مرة أخرى' });
    } finally {
      setSubmitting(false);
    }
  };

  const getFilteredRecords = (day, filterStatus) => {
    return records.filter(r => r.day == day && r.status === filterStatus);
  };

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-stone-50">
        <div className="text-center">
          <Loader2 className="w-10 h-10 text-emerald-600 animate-spin mx-auto mb-4"/>
          <p className="text-emerald-800">جاري الاتصال بقاعدة البيانات...</p>
          <p className="text-xs text-gray-400 mt-2">تأكد من إعداد Firebase بشكل صحيح</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-stone-50 font-sans text-right" dir="rtl">
      {/* 1. نافذة تأكيد الحذف */}
      {confirmModal.show && (
        <div className="fixed inset-0 bg-black/60 backdrop-blur-sm flex items-center justify-center z-[100] p-4 animate-fadeIn">
          <div className="bg-white rounded-2xl p-6 w-full max-w-sm shadow-2xl border-t-4 border-red-500 scale-100 transition-transform">
            <div className="flex flex-col items-center text-center mb-4">
              <div className="w-12 h-12 bg-red-100 text-red-600 rounded-full flex items-center justify-center mb-3">
                <AlertTriangle size={24} />
              </div>
              <h3 className="text-lg font-bold text-gray-900">تأكيد الحذف</h3>
              <p className="text-gray-600 mt-2 whitespace-pre-line">{confirmModal.message}</p>
            </div>
            <div className="flex gap-3">
              <Button onClick={confirmModal.onConfirm} variant="danger" className="flex-1 justify-center">نعم، احذف</Button>
              <Button variant="ghost" onClick={() => setConfirmModal({ ...confirmModal, show: false })} className="flex-1 justify-center">إلغاء</Button>
            </div>
          </div>
        </div>
      )}

      {/* 2. نافذة دخول المشرف */}
      {showAdminLogin && (
        <div className="fixed inset-0 bg-black/60 backdrop-blur-sm flex items-center justify-center z-50 p-4 animate-fadeIn">
          <div className="bg-white rounded-2xl p-6 w-full max-w-sm shadow-2xl border-t-4 border-emerald-600">
            <h3 className="text-xl font-bold mb-4 text-center text-emerald-900 flex justify-center items-center gap-2"><Lock className="w-5 h-5"/> دخول المشرف</h3>
            <form onSubmit={verifyAdminPassword} className="space-y-4">
              <div>
                <input type="password" value={adminPasswordInput} onChange={(e) => setAdminPasswordInput(e.target.value)} className="w-full p-3 bg-gray-50 border border-gray-300 rounded-lg focus:ring-2 focus:ring-emerald-500 outline-none text-center text-lg tracking-widest" placeholder="أدخل كلمة المرور" autoFocus />
                {loginError && <p className="text-red-500 text-sm mt-2 text-center">{loginError}</p>}
              </div>
              <div className="flex gap-2">
                <Button type="submit" className="flex-1">دخول</Button>
                <Button variant="ghost" onClick={() => {setShowAdminLogin(false); setLoginError(''); setAdminPasswordInput('');}} type="button">إلغاء</Button>
              </div>
            </form>
          </div>
        </div>
      )}

      {/* Header */}
      <header className="bg-emerald-700 text-white p-6 shadow-lg relative">
        <div className="max-w-5xl mx-auto flex justify-between items-center relative z-10">
          <div>
            <h1 className="text-3xl font-bold flex items-center gap-3"><BookOpen className="w-8 h-8 text-amber-300" />مُنجز رمضان</h1>
            <p className="text-emerald-100 mt-1 mr-11">تتبع وردك اليومي</p>
          </div>
          <button onClick={handleAdminAccess} className={`text-sm px-3 py-1 rounded-full transition-colors flex items-center gap-2 ${isAdmin ? 'bg-amber-400 text-emerald-900 font-bold shadow-lg' : 'bg-emerald-800 hover:bg-emerald-900 text-white'}`}>
            {isAdmin ? <LogOut size={16}/> : <Lock size={16}/>} {isAdmin ? 'خروج' : 'الإدارة'}
          </button>
        </div>
        <div className="absolute inset-0 opacity-10 pointer-events-none bg-[url('https://www.transparenttextures.com/patterns/arabesque.png')]"></div>
      </header>

      <main className="max-w-5xl mx-auto p-4 md:p-8">
        {isAdmin ? (
          <div className="space-y-6">
            <div className="flex gap-2 overflow-x-auto pb-2 scrollbar-hide">
              <Button variant={activeTab === 'form' ? 'primary' : 'ghost'} onClick={() => setActiveTab('form')}>تسجيل جديد</Button>
              <Button variant={activeTab === 'days' ? 'primary' : 'ghost'} onClick={() => setActiveTab('days')}>إدارة الأيام</Button>
              <Button variant={activeTab === 'students' ? 'primary' : 'ghost'} onClick={() => setActiveTab('students')}>الطلاب ({students.length})</Button>
              <Button variant={activeTab === 'data' ? 'primary' : 'ghost'} onClick={() => setActiveTab('data')}>لوحة المتابعة</Button>
            </div>

            {activeTab === 'days' && (
              <Card className="p-6 animate-fadeIn">
                <h2 className="text-xl font-bold mb-4 text-emerald-800 flex items-center gap-2"><CalendarClock className="w-6 h-6"/> التحكم بالأيام المتاحة</h2>
                <div className="grid grid-cols-5 md:grid-cols-6 lg:grid-cols-10 gap-3">
                  {fullDays.map(day => {
                    const isActive = activeDays.includes(day);
                    return (
                      <button key={day} onClick={() => toggleDayAvailability(day)} className={`relative p-3 rounded-xl font-bold text-lg transition-all duration-200 border-2 flex flex-col items-center justify-center gap-1 ${isActive ? 'bg-emerald-50 border-emerald-500 text-emerald-800 shadow-sm hover:bg-emerald-100' : 'bg-gray-50 border-gray-200 text-gray-400 hover:bg-gray-100 hover:border-gray-300'}`}>
                        <span>{day}</span>
                        {isActive ? <ToggleRight size={18} className="text-emerald-600"/> : <ToggleLeft size={18} className="text-gray-300"/>}
                      </button>
                    )
                  })}
                </div>
              </Card>
            )}

            {activeTab === 'students' && (
              <Card className="p-6 animate-fadeIn">
                <h2 className="text-xl font-bold mb-4 text-emerald-800 flex items-center gap-2"><Users className="w-5 h-5"/> إدارة الطلاب</h2>
                <div className="flex gap-2 mb-6">
                  <input type="text" value={newStudentName} onChange={(e) => setNewStudentName(e.target.value)} placeholder="اسم الطالب الجديد..." className="flex-1 p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-emerald-500 outline-none" />
                  <Button onClick={handleAddStudent}><Plus size={18}/> إضافة</Button>
                </div>
                <div className="bg-gray-50 rounded-lg border border-gray-200 max-h-96 overflow-y-auto">
                  {students.length === 0 ? <p className="p-4 text-center text-gray-500">لا يوجد طلاب مسجلين.</p> : (
                    <ul className="divide-y divide-gray-200">
                      {students.map((student) => (
                        <li key={student.id} className="relative bg-white p-3 flex justify-between items-center hover:bg-white transition-colors">
                          <span className="font-medium text-gray-700">{student.name}</span>
                          <button type="button" onClick={(e) => { e.stopPropagation(); requestDeleteStudent(student.id, student.name); }} className="relative z-10 flex items-center justify-center w-8 h-8 text-red-500 bg-red-50 hover:bg-red-100 hover:text-red-700 rounded-full transition-all cursor-pointer active:scale-90 border border-red-100 shadow-sm" title="حذف الطالب"><Trash2 size={18} className="pointer-events-none"/></button>
                        </li>
                      ))}
                    </ul>
                  )}
                </div>
              </Card>
            )}

            {activeTab === 'data' && (
              <div className="animate-fadeIn space-y-4">
                 <Card className="p-4 bg-emerald-800 text-white border-none">
                    <div className="flex flex-col md:flex-row justify-between items-center gap-4">
                      <div className="flex items-center gap-2"><BarChart className="w-6 h-6 text-amber-300"/><h2 className="text-xl font-bold">جرد الإنجاز اليومي</h2></div>
                      <div className="flex items-center bg-emerald-900/50 p-1 rounded-lg overflow-x-auto max-w-full w-full md:w-auto">
                        {fullDays.map(d => (
                          <button key={d} onClick={() => setViewDay(d.toString())} className={`px-3 py-1.5 rounded-md text-sm whitespace-nowrap transition-all ${viewDay === d.toString() ? 'bg-white text-emerald-900 font-bold shadow-sm' : 'text-emerald-100 hover:bg-emerald-700'}`}>{d} رمضان</button>
                        ))}
                      </div>
                    </div>
                 </Card>
                 <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
                    <StatusColumn title="قراءة وسماع" records={getFilteredRecords(viewDay, 'both')} icon={<div className="flex"><BookOpen size={16}/><Headphones size={16}/></div>} headerColor="bg-emerald-100 text-emerald-800 border-emerald-200" onDeleteRequest={requestDeleteRecord} />
                    <StatusColumn title="قراءة فقط" records={getFilteredRecords(viewDay, 'reading')} icon={<BookOpen size={16}/>} headerColor="bg-blue-100 text-blue-800 border-blue-200" onDeleteRequest={requestDeleteRecord} />
                    <StatusColumn title="سماع فقط" records={getFilteredRecords(viewDay, 'listening')} icon={<Headphones size={16}/>} headerColor="bg-purple-100 text-purple-800 border-purple-200" onDeleteRequest={requestDeleteRecord} />
                    <StatusColumn title="لم أنجز" records={getFilteredRecords(viewDay, 'none')} icon={<XCircle size={16}/>} headerColor="bg-red-100 text-red-800 border-red-200" onDeleteRequest={requestDeleteRecord} />
                 </div>
              </div>
            )}
          </div>
        ) : null}

        {(!isAdmin || activeTab === 'form') && (
          <div className="animate-fadeIn mt-6 grid grid-cols-1 lg:grid-cols-3 gap-6 items-start">
            <div className="lg:col-span-2 order-2 lg:order-1">
              <Card className="p-6 md:p-8">
                <div className="text-center mb-8"><h2 className="text-2xl font-bold text-gray-800 mb-2">تسجيل الإنجاز اليومي</h2><p className="text-gray-500">"قليل دائم خير من كثير منقطع"</p></div>
                {message && <div className={`mb-6 p-4 rounded-lg flex items-center gap-2 ${message.type === 'success' ? 'bg-emerald-100 text-emerald-800' : 'bg-red-100 text-red-800'}`}>{message.type === 'success' ? <CheckCircle size={20}/> : <XCircle size={20}/>}{message.text}</div>}
                <form onSubmit={handleSubmit} className="space-y-6">
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-2 flex items-center gap-2"><Users size={18} className="text-emerald-600"/> اختر الاسم</label>
                    {students.length === 0 ? <div className="text-center p-4 border-2 border-dashed border-gray-300 rounded-xl bg-gray-50 text-gray-500">يجب إضافة أسماء الطلاب من لوحة الإدارة أولاً</div> : (
                      <select value={selectedStudent} onChange={(e) => setSelectedStudent(e.target.value)} className="w-full p-3 bg-gray-50 border border-gray-300 rounded-xl focus:ring-2 focus:ring-emerald-500 focus:border-emerald-500 outline-none transition-all" required>
                        <option value="">-- اختر اسمك من القائمة --</option>
                        {students.map((s) => <option key={s.id} value={s.name}>{s.name}</option>)}
                      </select>
                    )}
                  </div>
                  <div>
                     <label className="block text-sm font-medium text-gray-700 mb-2 flex items-center gap-2"><Calendar size={18} className="text-emerald-600"/> اختر اليوم</label>
                    {activeDays.length === 0 ? <div className="text-center p-3 bg-red-50 text-red-600 rounded-lg border border-red-200">عفواً، لا توجد أيام متاحة للتسجيل حالياً.</div> : (
                      <select value={selectedDay} onChange={(e) => setSelectedDay(e.target.value)} className="w-full p-3 bg-gray-50 border border-gray-300 rounded-xl focus:ring-2 focus:ring-emerald-500 focus:border-emerald-500 outline-none transition-all" required>
                        <option value="">-- أي يوم من رمضان؟ --</option>
                        {activeDays.sort((a,b)=>a-b).map(d => <option key={d} value={d}>{d} رمضان</option>)}
                      </select>
                    )}
                  </div>
                  <div>
                     <label className="block text-sm font-medium text-gray-700 mb-3 flex items-center gap-2"><CheckCircle size={18} className="text-emerald-600"/> هل أنجزت المقدار؟</label>
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-3">
                      {[
                        { val: 'both', label: 'أنجزت القراءة والسماع', icon: <div className="flex"><BookOpen size={16}/><Headphones size={16}/></div> },
                        { val: 'reading', label: 'أنجزت القراءة فقط', icon: <BookOpen size={16}/> },
                        { val: 'listening', label: 'أنجزت السماع فقط', icon: <Headphones size={16}/> },
                        { val: 'none', label: 'لم أنجز اليوم', icon: <XCircle size={16}/> },
                      ].map((opt) => (
                        <label key={opt.val} className={`cursor-pointer p-3 rounded-xl border-2 flex items-center justify-between transition-all ${status === opt.val ? 'border-emerald-500 bg-emerald-50 text-emerald-900 shadow-sm' : 'border-gray-200 hover:border-emerald-200 hover:bg-gray-50 text-gray-600'}`}>
                          <div className="flex items-center gap-2"><input type="radio" name="status" value={opt.val} checked={status === opt.val} onChange={(e) => setStatus(e.target.value)} className="w-4 h-4 text-emerald-600 focus:ring-emerald-500"/><span className="font-medium text-sm">{opt.label}</span></div>
                          <div className={`${status === opt.val ? 'text-emerald-600' : 'text-gray-400'}`}>{opt.icon}</div>
                        </label>
                      ))}
                    </div>
                  </div>
                  <Button type="submit" disabled={submitting || students.length === 0 || activeDays.length === 0} className="w-full py-3 text-lg shadow-lg shadow-emerald-200">{submitting ? <Loader2 className="animate-spin"/> : <Save size={20} />} تسجيل الإنجاز</Button>
                </form>
              </Card>
            </div>
            <div className="lg:col-span-1 order-1 lg:order-2 h-full"><Leaderboard records={records} /></div>
          </div>
        )}
      </main>
      <footer className="text-center text-emerald-800/60 py-8 text-sm">تم التطوير للمساعدة على الطاعة في شهر الخير</footer>
    </div>
  );
}
