<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ตุ๊ดซี่เกม - ระบบลงทะเบียน</title>
    <!-- Tailwind CSS for styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React Libraries -->
    <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
    <!-- Babel for JSX compilation -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Lucide Icons -->
    <script type="module">
      import { Trash2, CheckCircle, UserPlus, Home, Tag, Mail, MapPin, X } from 'https://cdn.jsdelivr.net/npm/lucide-react@0.417.0/+esm'
      window.Lucide = { Trash2, CheckCircle, UserPlus, Home, Tag, Mail, MapPin, X };
    </script>
    <!-- Firebase Libraries -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js";
        import { getFirestore, collection, query, orderBy, onSnapshot, doc, addDoc, deleteDoc, setLogLevel } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js";
        
        window.firebaseImports = {
            initializeApp, getAuth, signInAnonymously, signInWithEmailAndPassword, onAuthStateChanged, signOut,
            getFirestore, collection, query, orderBy, onSnapshot, doc, addDoc, deleteDoc, setLogLevel
        };
    </script>

    <style>
        /* Simple animation for success view */
        @keyframes bounce-in {
            0% { transform: scale(0.8); opacity: 0; }
            50% { transform: scale(1.05); opacity: 1; }
            100% { transform: scale(1); }
        }
        .animate-bounce-in {
            animation: bounce-in 0.5s ease-out;
        }
    </style>
</head>
<body class="min-h-screen bg-gray-100 p-0 m-0">
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useCallback } = React;
        const { initializeApp, getAuth, signInAnonymously, onAuthStateChanged, getFirestore, collection, query, orderBy, onSnapshot, doc, addDoc, deleteDoc, setLogLevel } = window.firebaseImports;
        const { Trash2, CheckCircle, UserPlus, Home, Tag, Mail, MapPin, X } = window.Lucide;
        
        // =====================================================================
        // *** 🔴 ขั้นตอนที่ 2: แทนที่ด้วย Firebase Config ของคุณเองที่นี่ 🔴 ***
        // =====================================================================
        const FIREBASE_CONFIG = {
            // ตัวอย่างค่าที่คุณได้จากขั้นตอนที่ 1 (ห้ามใช้ค่าเหล่านี้)
            apiKey: "AIzaSyC...", 
            authDomain: "your-project-id.firebaseapp.com",
            projectId: "your-project-id",
            storageBucket: "your-project-id.appspot.com",
            messagingSenderId: "...",
            appId: "1:..."
        };

        // =====================================================================
        // *** 🔴 ขั้นตอนที่ 3: กำหนด ADMIN KEY สำหรับการลบ 🔴 ***
        // =====================================================================
        // กำหนด Key ลับสำหรับ Admin (ใช้ Key นี้แทน Token เก่า)
        // คุณต้องป้อน Key นี้ในช่อง Login เล็กๆ ที่จะปรากฏด้านล่าง
        const ADMIN_SECRET_KEY = "TOOTSIELOVE"; // เปลี่ยน Key นี้ให้เป็น Key ลับของคุณ!

        // =====================================================================
        // --- โค้ดที่เหลือจากเดิม (ไม่จำเป็นต้องแก้ไข) ---
        // =====================================================================

        const MAX_PARTICIPANTS = 200;

        // Path to the public collection (ใช้ projectId ของคุณเป็น appId แทน)
        const getPublicCollectionPath = (projectId) => 
            `artifacts/${projectId}/public/data/tootsie_game_registrations`;

        // Utility function to get Firebase instances
        const useFirebase = () => {
            const [db, setDb] = useState(null);
            const [auth, setAuth] = useState(null);
            const [currentUser, setCurrentUser] = useState(null);
            const [isAuthReady, setIsAuthReady] = useState(false);
            const [isAdmin, setIsAdmin] = useState(false); // New state for admin status

            useEffect(() => {
                if (!FIREBASE_CONFIG || !FIREBASE_CONFIG.projectId) {
                    console.error("Firebase config is missing or incomplete.");
                    return;
                }

                try {
                    setLogLevel('debug'); 
                    const app = initializeApp(FIREBASE_CONFIG);
                    const firestoreDb = getFirestore(app);
                    const firebaseAuth = getAuth(app);
                    
                    setDb(firestoreDb);
                    setAuth(firebaseAuth);

                    // Sign in anonymously for public read/write access
                    const authenticate = async () => {
                        try {
                            await signInAnonymously(firebaseAuth);
                            console.log("Signed in anonymously (Public access).");
                        } catch (error) {
                            console.error("Firebase authentication failed:", error);
                        }
                    };
                    authenticate();

                    // Set Auth State Listener
                    const unsubscribe = onAuthStateChanged(firebaseAuth, (user) => {
                        setCurrentUser(user);
                        setIsAuthReady(true);
                        // Admin status will be checked separately via the login form
                    });

                    return () => unsubscribe();
                } catch (error) {
                    console.error("Failed to initialize Firebase:", error);
                }
            }, []);

            return { db, auth, currentUser, isAuthReady, isAdmin, setIsAdmin };
        };

        const AdminLogin = ({ isAdmin, setIsAdmin }) => {
            const [key, setKey] = useState('');
            const [error, setError] = useState('');

            const handleLogin = () => {
                setError('');
                if (key === ADMIN_SECRET_KEY) {
                    setIsAdmin(true);
                    sessionStorage.setItem('isAdmin', 'true'); // Persist admin status
                    setKey('');
                } else {
                    setError('Admin Key ผิดจ้า');
                }
            };
            
            // Check session storage on load
            useEffect(() => {
                if (sessionStorage.getItem('isAdmin') === 'true') {
                    setIsAdmin(true);
                }
            }, []);

            if (isAdmin) {
                return <p className="text-sm font-bold text-red-600 bg-red-100 p-2 rounded-lg mt-3">✅ สถานะ: ADMIN (สามารถลบรายชื่อได้)</p>;
            }

            return (
                <div className="flex justify-center items-center mt-3 space-x-2 p-2 bg-gray-50 rounded-lg shadow-inner max-w-lg mx-auto">
                    <input 
                        type="password" 
                        placeholder="ป้อน Admin Key" 
                        value={key} 
                        onChange={(e) => setKey(e.target.value)} 
                        className="px-2 py-1 border rounded-md text-sm w-36"
                    />
                    <button 
                        onClick={handleLogin} 
                        className="bg-purple-500 text-white text-sm px-3 py-1 rounded-md hover:bg-purple-600 transition-colors"
                    >
                        Login Admin
                    </button>
                    {error && <span className="text-red-500 text-xs ml-2">{error}</span>}
                </div>
            );
        };

        // --- Form Components ---

        const RegistrationForm = ({ db, currentUser, onRegisterSuccess, currentCount }) => {
            const [formData, setFormData] = useState({
                name: '',
                nickname: '',
                house_number: '',
                postal_code: '',
                province: '',
                address_extra: ''
            });
            const [isLoading, setIsLoading] = useState(false);
            const [error, setError] = useState(null);

            const handleChange = (e) => {
                setFormData({ ...formData, [e.target.name]: e.target.value });
            };

            const handleSubmit = async (e) => {
                e.preventDefault();
                setError(null);
                
                if (!db || !currentUser) {
                    setError("ระบบยังไม่พร้อม โปรดรอสักครู่");
                    return;
                }

                if (currentCount >= MAX_PARTICIPANTS) {
                    setError(`เสียใจด้วยค่ะ จำนวนผู้สมัครเต็มแล้ว (${MAX_PARTICIPANTS} คน)`);
                    return;
                }

                const requiredFields = ['name', 'nickname', 'house_number', 'postal_code', 'province', 'address_extra'];
                const isValid = requiredFields.every(field => formData[field].trim() !== '');
                if (!isValid) {
                    setError("กรุณากรอกข้อมูลให้ครบทุกช่อง");
                    return;
                }

                setIsLoading(true);
                try {
                    const registrationData = {
                        ...formData,
                        timestamp: Date.now(),
                        userId: currentUser.uid,
                    };

                    const collectionRef = collection(db, getPublicCollectionPath(FIREBASE_CONFIG.projectId));
                    await addDoc(collectionRef, registrationData);

                    onRegisterSuccess();
                } catch (err) {
                    console.error("Error adding document: ", err);
                    setError("เกิดข้อผิดพลาดในการลงทะเบียน โปรดลองอีกครั้ง");
                } finally {
                    setIsLoading(false);
                }
            };

            const inputClasses = "w-full p-3 border border-pink-300 rounded-lg focus:ring-2 focus:ring-pink-500 transition-all text-gray-800";
            const labelClasses = "flex items-center text-sm font-medium text-pink-700 mb-1";
            const buttonClasses = "w-full py-3 mt-4 bg-pink-600 text-white font-bold rounded-xl shadow-lg hover:bg-pink-700 transition-all duration-300 transform hover:scale-[1.01] disabled:opacity-60";

            return (
                <div className="p-6 bg-white rounded-2xl shadow-2xl border-4 border-pink-400 max-w-lg w-full mx-auto">
                    <h2 className="text-3xl font-extrabold text-center text-pink-700 mb-2 flex items-center justify-center">
                        <UserPlus className="w-6 h-6 mr-2"/>
                        ลงทะเบียน "ตุ๊ดซี่เกม"
                    </h2>
                    <p className="text-center text-gray-500 mb-6">
                        จำนวนผู้สมัคร: <span className="font-bold text-pink-600">{currentCount} / {MAX_PARTICIPANTS}</span>
                    </p>
                    {error && (
                        <div className="p-3 mb-4 bg-red-100 border border-red-400 text-red-700 rounded-lg">
                            {error}
                        </div>
                    )}
                    
                    <form onSubmit={handleSubmit} className="space-y-4">
                        {/* Name and Nickname */}
                        <div className="flex space-x-2">
                            <div className="w-1/2">
                                <label className={labelClasses} htmlFor="name"><Tag className="w-4 h-4 mr-1"/>ชื่อจริง</label>
                                <input type="text" id="name" name="name" value={formData.name} onChange={handleChange} className={inputClasses} placeholder="ชื่อ-นามสกุล" required />
                            </div>
                            <div className="w-1/2">
                                <label className={labelClasses} htmlFor="nickname"><Tag className="w-4 h-4 mr-1"/>ชื่อเล่น</label>
                                <input type="text" id="nickname" name="nickname" value={formData.nickname} onChange={handleChange} className={inputClasses} placeholder="ชื่อเล่น" required />
                            </div>
                        </div>

                        {/* Address Details */}
                        <div className="space-y-4 p-4 border border-pink-200 rounded-xl bg-pink-50">
                            <h3 className="font-bold text-pink-600 flex items-center"><Home className="w-4 h-4 mr-1"/> ข้อมูลที่อยู่</h3>
                            
                            <div>
                                <label className={labelClasses} htmlFor="house_number">บ้านเลขที่</label>
                                <input type="text" id="house_number" name="house_number" value={formData.house_number} onChange={handleChange} className={inputClasses} placeholder="บ้านเลขที่" required />
                            </div>
                            
                            <div className="flex space-x-2">
                                <div className="w-1/2">
                                    <label className={labelClasses} htmlFor="province">จังหวัด</label>
                                    <input type="text" id="province" name="province" value={formData.province} onChange={handleChange} className={inputClasses} placeholder="เช่น กรุงเทพฯ" required />
                                </div>
                                <div className="w-1/2">
                                    <label className={labelClasses} htmlFor="postal_code">รหัสไปรษณีย์</label>
                                    <input type="text" id="postal_code" name="postal_code" value={formData.postal_code} onChange={handleChange} className={inputClasses} pattern="\d{5}" maxLength="5" placeholder="5 หลัก" required />
                                </div>
                            </div>
                            
                            <div>
                                <label className={labelClasses} htmlFor="address_extra"><MapPin className="w-4 h-4 mr-1"/>ซอย/ตำบล/อำเภอ/หมู่ (เพิ่มเติม)</label>
                                <textarea id="address_extra" name="address_extra" value={formData.address_extra} onChange={handleChange} rows="2" className={`${inputClasses} resize-none`} placeholder="ซอย, ตำบล, อำเภอ, หมู่" required />
                            </div>
                        </div>

                        <button type="submit" disabled={isLoading || currentCount >= MAX_PARTICIPANTS} className={buttonClasses}>
                            {isLoading ? 'กำลังส่งข้อมูล...' : 'ลงทะเบียนเข้าร่วมกิจกรรม'}
                        </button>
                    </form>
                </div>
            );
        };

        const SuccessView = ({ onBackToForm }) => {
            return (
                <div className="p-8 bg-gradient-to-br from-pink-100 to-red-100 rounded-3xl shadow-2xl border-4 border-pink-500 max-w-lg w-full mx-auto text-center animate-bounce-in">
                    <CheckCircle className="w-16 h-16 text-pink-600 mx-auto mb-4" />
                    <h2 className="text-4xl font-extrabold text-pink-800 mb-2">สำเร็จ!</h2>
                    <p className="text-2xl font-bold text-pink-700">
                        "ขอให้โชคดีอย่าให้โดนปัดตุ๊บจ้า"
                    </p>
                    <p className="text-gray-600 mt-4">
                        ดูรายชื่อผู้สมัครด้านล่างได้เลยค่ะ
                    </p>
                    <button 
                        onClick={onBackToForm}
                        className="mt-6 px-6 py-2 bg-pink-500 text-white font-medium rounded-full shadow-md hover:bg-pink-600 transition-colors"
                    >
                        กลับไปหน้าหลัก
                    </button>
                </div>
            );
        };

        // --- Modal Component for Details ---

        const RegistrationDetailModal = ({ registration, onClose, onDelete, isAdmin }) => {
            if (!registration) return null;

            const DetailItem = ({ icon: Icon, label, value }) => (
                <div className="flex items-start text-gray-700 mb-2">
                    <Icon className="w-5 h-5 text-purple-500 mr-3 mt-1 flex-shrink-0" />
                    <div>
                        <span className="font-bold text-sm text-purple-700 block">{label}</span>
                        <span className="text-base text-gray-800 break-words block">{value}</span>
                    </div>
                </div>
            );

            return (
                <div className="fixed inset-0 bg-black bg-opacity-50 flex justify-center items-center z-50 p-4" onClick={onClose}>
                    <div 
                        className="bg-white rounded-xl w-full max-w-md p-6 shadow-2xl border-4 border-purple-500 relative transform transition-all duration-300 scale-100 opacity-100"
                        onClick={e => e.stopPropagation()} // Prevent closing when clicking inside modal
                    >
                        <button 
                            onClick={onClose} 
                            className="absolute top-3 right-3 p-1 rounded-full text-gray-400 hover:bg-gray-100 hover:text-gray-600 transition-colors"
                            aria-label="ปิด"
                        >
                            <X className="w-6 h-6" />
                        </button>

                        <h3 className="text-2xl font-extrabold text-purple-700 mb-4 border-b pb-2">
                            ข้อมูลผู้สมัครโดยละเอียด
                        </h3>

                        <div className="space-y-4">
                            <div className="flex space-x-4">
                                <div className="w-1/2">
                                    <DetailItem icon={Tag} label="ชื่อจริง" value={registration.name} />
                                </div>
                                <div className="w-1/2">
                                    <DetailItem icon={Tag} label="ชื่อเล่น" value={registration.nickname} />
                                </div>
                            </div>
                            
                            <h4 className="text-lg font-bold text-pink-600 mt-4 flex items-center"><Home className="w-5 h-5 mr-2"/>ข้อมูลที่อยู่</h4>
                            
                            <DetailItem icon={MapPin} label="บ้านเลขที่" value={registration.house_number} />
                            <DetailItem icon={Mail} label="รหัสไปรษณีย์" value={registration.postal_code} />
                            <DetailItem icon={MapPin} label="จังหวัด" value={registration.province} />
                            <DetailItem icon={MapPin} label="รายละเอียดเพิ่มเติม (ซอย/ตำบล/อำเภอ/หมู่)" value={registration.address_extra} />
                            
                            {isAdmin && (
                                <div className="pt-4 border-t mt-4">
                                    <button 
                                        onClick={() => onDelete(registration.id)}
                                        className="w-full py-2 bg-red-500 text-white font-bold rounded-lg shadow-md hover:bg-red-600 transition-all disabled:opacity-50 flex items-center justify-center"
                                        title="ลบรายชื่อผู้สมัคร"
                                    >
                                        <Trash2 className="w-5 h-5 mr-2"/>
                                        {'ลบรายชื่อนี้ (สำหรับ Admin)'}
                                    </button>
                   
