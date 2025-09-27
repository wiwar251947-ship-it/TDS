import React, { useState, useEffect, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { 
    getFirestore, 
    collection, 
    query, 
    orderBy, 
    onSnapshot, 
    doc, 
    addDoc, 
    deleteDoc,
    setLogLevel
} from 'firebase/firestore';
import { Trash2, CheckCircle, UserPlus, Home, Tag, Mail, MapPin, X } from 'lucide-react';

// --- Firebase and App Setup ---

// Global variables provided by the environment
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// The maximum number of participants allowed
const MAX_PARTICIPANTS = 200;

// Path to the public collection
// This path ensures data is publicly readable and writable by anyone in this canvas environment
const getPublicCollectionPath = () => 
    `artifacts/${appId}/public/data/tootsie_game_registrations`;

// Utility function to get Firebase instances
const useFirebase = () => {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [currentUser, setCurrentUser] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    useEffect(() => {
        if (!firebaseConfig) {
            console.error("Firebase config is missing.");
            return;
        }

        try {
            // Set Firebase log level to Debug for visibility
            setLogLevel('debug'); 
            const app = initializeApp(firebaseConfig);
            const firestoreDb = getFirestore(app);
            const firebaseAuth = getAuth(app);
            
            setDb(firestoreDb);
            setAuth(firebaseAuth);

            // 1. Initial Authentication
            const authenticate = async () => {
                try {
                    if (initialAuthToken) {
                        // Sign in with provided custom token (usually for the "owner"/creator)
                        await signInWithCustomToken(firebaseAuth, initialAuthToken);
                        console.log("Signed in with custom token (Admin access granted).");
                    } else {
                        // Sign in anonymously (for general users/public access)
                        await signInAnonymously(firebaseAuth);
                        console.log("Signed in anonymously (Public access).");
                    }
                } catch (error) {
                    console.error("Firebase authentication failed:", error);
                }
            };
            authenticate();

            // 2. Auth State Listener
            const unsubscribe = onAuthStateChanged(firebaseAuth, (user) => {
                setCurrentUser(user);
                setIsAuthReady(true);
            });

            return () => unsubscribe(); // Cleanup auth listener
        } catch (error) {
            console.error("Failed to initialize Firebase:", error);
        }
    }, [initialAuthToken]);

    return { db, auth, currentUser, isAuthReady };
};

// --- Form Components ---

const RegistrationForm = ({ db, currentUser, onRegisterSuccess, currentCount }) => {
    const [formData, setFormData] = useState({
        name: '',
        nickname: '',
        phone_number: '', // <-- ADDED: Phone number state
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

        // ADDED 'phone_number' to required fields
        const requiredFields = ['name', 'nickname', 'phone_number', 'house_number', 'postal_code', 'province', 'address_extra'];
        const isValid = requiredFields.every(field => formData[field].trim() !== '');
        
        // Basic phone number validation (10 digits)
        const isPhoneValid = /^\d{10}$/.test(formData.phone_number.trim());

        if (!isValid) {
            setError("กรุณากรอกข้อมูลให้ครบทุกช่อง");
            return;
        }
        if (!isPhoneValid) {
            setError("กรุณากรอกเบอร์โทรศัพท์ให้ถูกต้อง 10 หลัก (เฉพาะตัวเลข)");
            return;
        }

        setIsLoading(true);
        try {
            const registrationData = {
                ...formData,
                timestamp: Date.now(),
                userId: currentUser.uid,
            };

            const collectionRef = collection(db, getPublicCollectionPath());
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

                {/* ADDED: Phone Number */}
                <div>
                    <label className={labelClasses} htmlFor="phone_number"><Mail className="w-4 h-4 mr-1"/>เบอร์โทรศัพท์มือถือ</label>
                    <input 
                        type="tel" 
                        id="phone_number" 
                        name="phone_number" 
                        value={formData.phone_number} 
                        onChange={handleChange} 
                        className={inputClasses} 
                        placeholder="08X-XXX-XXXX (10 หลัก)" 
                        pattern="[0-9]{10}"
                        maxLength="10"
                        required 
                    />
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
                    
                    {/* ADDED: Phone number detail display */}
                    <DetailItem icon={Mail} label="เบอร์โทรศัพท์มือถือ" value={registration.phone_number} />

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
                        </div>
                    )}
                </div>
            </div>
        </div>
    );
};

// --- List Component (Public/Admin View) ---

const RegistrationList = ({ db, auth, currentUser, isAuthReady, registrations, currentCount, onSelectRegistration, onDeleteRegistration }) => {
    // Check if the user is the creator/owner (assumed to have the initialAuthToken)
    const isAdmin = currentUser && initialAuthToken !== null; 

    // Function to handle row click
    const handleRowClick = (reg) => {
        onSelectRegistration(reg);
    };

    return (
        <div className="mt-12 p-6 bg-white rounded-2xl shadow-2xl border-4 border-purple-400 max-w-4xl w-full mx-auto">
            <h2 className="text-3xl font-extrabold text-center text-purple-700 mb-6">
                รายชื่อผู้สมัคร "ตุ๊ดซี่เกม" (สาธารณะ)
            </h2>
            <p className="text-center text-gray-600 mb-4">
                <span className="font-bold text-purple-600">{currentCount}</span> คน / จำกัด {MAX_PARTICIPANTS} คน
            </p>
            
            {!isAuthReady && (
                 <div className="text-center text-purple-500 py-4">กำลังโหลดข้อมูลผู้ใช้...</div>
            )}
            
            <div className="overflow-x-auto">
                {registrations.length === 0 ? (
                    <p className="text-center text-gray-500 py-8">
                        ยังไม่มีผู้สมัครเลยจ้า มาเป็นคนแรกเลย!
                    </p>
                ) : (
                    <table className="min-w-full divide-y divide-purple-200 rounded-xl overflow-hidden">
                        <thead className="bg-purple-100">
                            <tr>
                                <th className="px-3 py-3 text-left text-xs font-medium text-purple-600 uppercase tracking-wider">#</th>
                                <th className="px-3 py-3 text-left text-xs font-medium text-purple-600 uppercase tracking-wider">ชื่อจริง</th>
                                <th className="px-3 py-3 text-left text-xs font-medium text-purple-600 uppercase tracking-wider">ชื่อเล่น</th>
                                <th className="px-3 py-3 text-left text-xs font-medium text-purple-600 uppercase tracking-wider hidden sm:table-cell">ที่อยู่ (ย่อ)</th>
                                {isAdmin && (
                                    <th className="px-3 py-3 text-center text-xs font-medium text-purple-600 uppercase tracking-wider">ลบ</th>
                                )}
                            </tr>
                        </thead>
                        <tbody className="bg-white divide-y divide-purple-100">
                            {registrations.map((reg, index) => (
                                <tr 
                                    key={reg.id} 
                                    className="hover:bg-purple-50 transition-colors cursor-pointer"
                                    onClick={() => handleRowClick(reg)} // Open modal on row click
                                >
                                    <td className="px-3 py-3 whitespace-nowrap text-sm font-medium text-gray-900">{index + 1}</td>
                                    <td className="px-3 py-3 whitespace-nowrap text-sm text-gray-700">{reg.name}</td>
                                    <td className="px-3 py-3 whitespace-nowrap text-sm font-bold text-pink-600">{reg.nickname}</td>
                                    <td className="px-3 py-3 whitespace-normal text-sm text-gray-500 hidden sm:table-cell">
                                        {reg.house_number} ({reg.postal_code}), {reg.province}
                                    </td>
                                    {isAdmin && (
                                        <td className="px-3 py-3 whitespace-nowrap text-center" onClick={(e) => e.stopPropagation()}> 
                                            {/* Clicking delete in the table will still prompt for confirmation and use the onDeleteRegistration function */}
                                            <button 
                                                onClick={() => onDeleteRegistration(reg.id)}
                                                className="p-1 text-red-500 hover:text-red-700 disabled:opacity-50 transition-colors rounded-full hover:bg-red-100"
                                                title="ลบรายชื่อผู้สมัคร"
                                            >
                                                <Trash2 className="w-5 h-5"/>
                                            </button>
                                        </td>
                                    )}
                                </tr>
                            ))}
                        </tbody>
                    </table>
                )}
            </div>
        </div>
    );
};

// --- Main App Component ---

const App = () => {
    const { db, auth, currentUser, isAuthReady } = useFirebase();
    const [view, setView] = useState('form'); /
