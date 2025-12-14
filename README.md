import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, collection } from 'firebase/firestore';

// --- FIREBASE CONFIGURATION & GLOBALS ---
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// --- TIME UTILITIES ---

// Helper function to convert HH:MM to Date object for today
const getPrayerDate = (prayerTime) => {
    const now = new Date();
    const [hour, minute] = prayerTime.split(':').map(Number);
    // Note: Creating date object for today's date at prayer time
    return new Date(now.getFullYear(), now.getMonth(), now.getDate(), hour, minute, 0);
};

// Check if the current time is past the prayer time
const isTimePassed = (prayerTime) => {
    // Add a small buffer (e.g., 60 seconds) to avoid issues right on the minute boundary
    return new Date().getTime() > getPrayerDate(prayerTime).getTime() + (60 * 1000);
};

// Check if the current time is within 10 minutes BEFORE the prayer time (Early Completion Window)
const isWithinEarlyWindow = (prayerTime) => {
    const nowMs = new Date().getTime();
    const adzanTimeMs = getPrayerDate(prayerTime).getTime();
    // Calculate 10 minutes before Adzan in milliseconds
    const tenMinutesBeforeAdzanMs = adzanTimeMs - (10 * 60 * 1000); 

    // True if current time is before Adzan AND after or equal to 10 minutes before Adzan
    return nowMs < adzanTimeMs && nowMs >= tenMinutesBeforeAdzanMs;
};

// Function to format HH:MM to H:MM AM/PM (12-hour format for display)
// Output: HH:MM pagi/sore
const formatTime12h = (time24h) => {
    if (!time24h) return 'XX:XX';
    const [h, m] = time24h.split(':').map(Number);
    // Use a fixed date to correctly parse time
    const date = new Date(2000, 0, 1, h, m);
    // This provides the HH:MM format (e.g., 4:35 AM)
    return date.toLocaleTimeString('en-US', { 
        hour: 'numeric', 
        minute: '2-digit', 
        hour12: true 
    }).replace(' AM', ' pagi').replace(' PM', ' sore'); // Simplified AM/PM for Indonesian context
};

// --- QIBLA UTILITIES ---

/**
 * Calculates the Qibla bearing (angle from True North) using the Great Circle method.
 * @param {number} userLat - User's Latitude
 * @param {number} userLon - User's Longitude
 * @returns {number} Bearing in degrees (0 to 360)
 */
const calculateQiblaAngle = (userLat, userLon) => {
    const KAABA_LAT = 21.4225; // Mecca Latitude
    const KAABA_LON = 39.8262; // Mecca Longitude

    // Convert degrees to radians
    const latRad = userLat * (Math.PI / 180);
    const kaabaLatRad = KAABA_LAT * (Math.PI / 180);
    const deltaLonRad = (KAABA_LON - userLon) * (Math.PI / 180);

    const y = Math.sin(deltaLonRad) * Math.cos(kaabaLatRad);
    const x = Math.cos(latRad) * Math.sin(kaabaLatRad) -
              Math.sin(latRad) * Math.cos(kaabaLatRad) * Math.cos(deltaLonRad);

    let bearing = Math.atan2(y, x) * (180 / Math.PI);
    
    // Convert bearing from -180 to 180 to 0 to 360 (clockwise from North)
    if (bearing < 0) {
        bearing += 360;
    }

    return bearing; 
};


// --- HAID/MOON UTILITIES ---

const getHaidStatus = (haidData) => {
    const defaultStatus = { isHaid: false, haidDay: 0, totalDays: 0 };
    if (!haidData || !haidData.startDate || !haidData.endDate) {
        return defaultStatus;
    }

    const today = new Date();
    today.setHours(0, 0, 0, 0);

    // Convert stored ISO string dates to Date objects set to midnight
    const startDate = new Date(haidData.startDate);
    startDate.setHours(0, 0, 0, 0);

    const endDate = new Date(haidData.endDate);
    endDate.setHours(0, 0, 0, 0);

    // Ensure the dates are valid and start <= end
    if (startDate > endDate || isNaN(startDate.getTime()) || isNaN(endDate.getTime())) {
        return defaultStatus;
    }

    // Check if today is within the range [start, end]
    if (today.getTime() >= startDate.getTime() && today.getTime() <= endDate.getTime()) {
        const msPerDay = 1000 * 60 * 60 * 24;
        
        // Calculate the difference in days from the start date (Haid Day number)
        const diffTime = today.getTime() - startDate.getTime();
        const haidDay = Math.floor(diffTime / msPerDay) + 1; 

        // Calculate total duration (for "H1 of 7", etc.)
        const totalDurationMs = endDate.getTime() - startDate.getTime();
        const totalDays = Math.floor(totalDurationMs / msPerDay) + 1;

        return {
            isHaid: true,
            haidDay: haidDay, // H1, H2, H3...
            totalDays: totalDays
        };
    }

    return defaultStatus;
};


// --- ICON COMPONENTS ---
const LocationIcon = ({ className }) => (
    <svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z"/><circle cx="12" cy="10" r="3"/></svg>
);
const HomeIcon = ({ className }) => (
    <svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M3 9l9-7 9 9v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/><polyline points="9 22 9 12 15 12 15 22"/></svg>
);
const CheckCircleIcon = ({ className }) => (
    <svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>
);
const QuranIcon = ({ className }) => (
    <svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M14 2v10l-2-2-2 2V2"/><path d="M5 2h14c1.1 0 2 .9 2 2v16c0 1.1-.9 2-2 2H5c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2z"/><path d="M12 16h.01"/></svg>
);
const SettingsIcon = ({ className }) => (
    <svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12.22 2h-.44a2 2 0 0 0-2 2v.18a2 2 0 0 1-1 1.73l-.43.25a2 2 0 0 1-2 0l-.15-.08a2 2 0 0 0-2.73.73l-.78 1.35a2 2 0 0 0 .73 2.73l.15.08a2 2 0 0 1 1 1.73v.54a2 2 0 0 1-1 1.73l-.15.08a2 2 0 0 0-.73 2.73l.78 1.35a2 2 0 0 0 2.73.73l.15-.08a2 2 0 0 1 2 0l.43.25a2 2 0 0 1 1 1.73V20a2 2 0 0 0 2 2h.44a2 2 0 0 0 2-2v-.18a2 2 0 0 1 1-1.73l.43-.25a2 2 0 0 1 2 0l.15.08a2 2 0 0 0 2.73-.73l.78-1.35a2 2 0 0 0-.73-2.73l-.15-.08a2 2 0 0 1-1-1.73v-.54a2 2 0 0 1 1-1.73l-.15-.08a2 2 0 0 0-.73-2.73l-.78-1.35a2 2 0 0 0-2.73-.73l-.15.08a2 2 0 0 1-2 0l-.43-.25a2 2 0 0 1-1-1.73V4a2 2 0 0 0-2-2z"/><circle cx="12" cy="12" r="3"/></svg>
);
const MoonIcon = ({ className }) => (
    <svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 3a6 6 0 0 0 9 9 9 9 0 1 1-9-9Z"/></svg>
);
const SparkleIcon = ({ className }) => (
    <svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 2v2"/><path d="M12 20v2"/><path d="M4.93 4.93l1.41 1.41"/><path d="M17.66 17.66l1.41 1.41"/><path d="M2 12h2"/><path d="M20 12h2"/><path d="M4.93 19.07l1.41-1.41"/><path d="M17.66 6.34l1.41-1.41"/><circle cx="12" cy="12" r="3"/></svg>
);
// NEW: Compass Icon for Qibla
const CompassIcon = ({ className }) => (
    <svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 21a9 9 0 0 0-9-9 9 9 0 0 0 9-9 9 9 0 0 0 9 9 9 9 0 0 0-9 9"/><path d="m14 10-2 2-2-2"/><path d="M12 21V12"/><path d="M12 12h9"/></svg>
);


// New Thematic Background for the Active Prayer Card (Desert/Warm Tones)
const ThematicBackground = () => (
    <svg 
        className="absolute bottom-0 right-0 w-full h-full opacity-100"
        viewBox="0 0 1000 700" 
        preserveAspectRatio="xMidYMax slice"
        xmlns="http://www.w3.org/2000/svg"
        fill="none"
    >
        <defs>
            <linearGradient id="desertGradient" x1="0%" y1="0%" x2="0%" y2="100%">
                <stop offset="0%" style={{stopColor: "#F59E0B", stopOpacity: 1}} />
                <stop offset="100%" style={{stopColor: "#FCD34D", stopOpacity: 1}} />
            </linearGradient>
            <linearGradient id="duneGradient" x1="0%" y1="0%" x2="0%" y2="100%">
                <stop offset="0%" style={{stopColor: "#C2410C", stopOpacity: 1}} />
                <stop offset="100%" style={{stopColor: "#F59E0B", stopOpacity: 1}} />
            </linearGradient>
        </defs>
        
        <rect x="0" y="0" width="1000" height="700" fill="url(#desertGradient)" />

        <circle cx="150" cy="100" r="70" fill="#FEF3C7" opacity="0.8"/> 

        <path d="M 0 700 V 550 C 150 500, 300 650, 500 600 L 500 700 Z" fill="#92400E" opacity="0.8" /> 
        
        <path d="M 400 700 V 500 C 600 400, 800 550, 1000 450 L 1000 700 Z" fill="url(#duneGradient)" opacity="1" />
        
        <g fill="#78350F" opacity="0.9"> 
            <path d="M 600 550 L 610 540 L 620 550 L 610 560 Z" />
            <path d="M 640 540 L 650 530 L 660 540 L 650 550 Z" />
        </g>
        
    </svg>
);


// --- QURAN DATA (30 Short Surahs from Juz 30) ---
const SHORT_SURAHS = [
    { id: 1, name: "Al-Fatihah", ayat: 7, arabic: "بِسْمِ ٱللَّهِ ٱلرَّحْمَٰنِ ٱلرَّحِيمِ", translation: "Pembukaan" },
    { id: 2, name: "An-Nas", ayat: 6, arabic: "قُلْ أَعُوذُ بِرَبِّ ٱلنَّاسِ", translation: "Manusia" },
    { id: 3, name: "Al-Falaq", ayat: 5, arabic: "قُلْ أَعُوذُ بِرَبِّ ٱلْفَلَقِ", translation: "Waktu Subuh" },
    { id: 4, name: "Al-Ikhlas", ayat: 4, arabic: "قُلْ هُوَ ٱللَّهُ أَحَدٌ", translation: "Memurnikan Keesaan Allah" },
    { id: 5, name: "Al-Masad", ayat: 5, arabic: "تَبَّتْ يَدَآ أَبِى لَهَبٍ وَتَبَّ", translation: "Gejolak Api (Abu Lahab)" },
    { id: 6, name: "An-Nasr", ayat: 3, arabic: "إِذَا جَآءَ نَصْرُ ٱللَّهِ وَٱلْفَتْحُ", translation: "Pertolongan" },
    { id: 7, name: "Al-Kafirun", ayat: 6, arabic: "قُلْ يَٰٓأَيُّهَا ٱلْكَٰفِرُونَ", translation: "Orang-orang Kafir" },
    { id: 8, name: "Al-Kautsar", ayat: 3, arabic: "إِنَّآ أَعْطَيْنَٰكَ ٱلْكَوْثَرَ", translation: "Nikmat yang Banyak" },
    { id: 9, name: "Al-Ma'un", ayat: 7, arabic: "أَرَءَيْتَ ٱلَّذِى يُكَذِّبُ بِٱلدِّينِ", translation: "Barang-barang yang Berguna" },
    { id: 10, name: "Quraisy", ayat: 4, arabic: "لِإِيلَٰفِ قُرَيْشٍ", translation: "Suku Quraisy" },
    { id: 11, name: "Al-Fil", ayat: 5, arabic: "أَلَمْ تَرَ كَيْفَ فَعَلَ رَبُّكَ بِأَصْحَٰبِ ٱلْفِيلِ", translation: "Gajah" },
    { id: 12, name: "Al-Humazah", ayat: 9, arabic: "وَيْلٌ لِّكُلِّ هُمَزَةٍ لُّمَزَةٍ", translation: "Pengumpat" },
    { id: 13, name: "Al-Asr", ayat: 3, arabic: "وَٱلْعَصْرِ", translation: "Masa" },
    { id: 14, name: "At-Takatsur", ayat: 8, arabic: "أَلْهَىٰكُمُ ٱلتَّكَاثُرُ", translation: "Bermegah-megahan" },
    { id: 15, name: "Al-Qari'ah", ayat: 11, arabic: "ٱلْقَارِعَةُ", translation: "Hari Kiamat" },
    { id: 16, name: "Al-'Adiyat", ayat: 11, arabic: "وَٱلْعَٰدِيَٰتِ ضَبْحًا", translation: "Kuda Perang yang Berlari Kencang" },
    { id: 17, name: "Az-Zalzalah", ayat: 8, arabic: "إِذَا زُلْزِلَتِ ٱلْأَرْضُ زِلْزَالَهَا", translation: "Goncangan" },
    { id: 18, name: "Al-Bayyinah", ayat: 8, arabic: "لَمْ يَكُنِ ٱلَّذِينَ كَفَرُوا۟ مِنْ أَهْلِ ٱلْكِتَٰبِ", translation: "Bukti Nyata" },
    { id: 19, name: "Al-Qadr", ayat: 5, arabic: "إِنَّآ أَنزَلْنَٰهُ فِى لَيْلَةِ ٱلْقَدْرِ", translation: "Kemuliaan" },
    { id: 20, name: "Al-'Alaq", ayat: 19, arabic: "ٱقْرَأْ بِٱسْمِ رَبِّكَ ٱلَّذِى خَلَقَ", translation: "Segumpal Darah" },
    { id: 21, name: "At-Tin", ayat: 8, arabic: "وَٱلتِّينِ وَٱلزَّيْتُونِ", translation: "Buah Tin" },
    { id: 22, name: "Asy-Syarh", ayat: 8, arabic: "أَلَمْ نَشْرَحْ لَكَ صَدْرَكَ", translation: "Kelapangan" },
    { id: 23, name: "Ad-Dhuha", ayat: 11, arabic: "وَٱلضُّحَىٰ", translation: "Waktu Dhuha" },
    { id: 24, name: "Al-Lail", ayat: 21, arabic: "وَٱلَّيْلِ إِذَا يَغْشَىٰ", translation: "Malam" },
    { id: 25, name: "Asy-Syams", ayat: 15, arabic: "وَٱلشَّمْسِ وَضُحَىٰهَا", translation: "Matahari" },
    { id: 26, name: "Al-Balad", ayat: 20, arabic: "لَآ أُقْسِمُ بِهَٰذَا ٱلْبَلَدِ", translation: "Negeri" },
    { id: 27, name: "Al-Fajr", ayat: 30, arabic: "وَٱلْفَجْرِ", translation: "Fajar" },
    { id: 28, name: "Al-Ghasyiyah", ayat: 26, arabic: "هَلْ أَتَىٰكَ حَدِيثُ ٱلْغَٰشِيَةِ", translation: "Hari Pembalasan" },
    { id: 29, name: "Al-A'la", ayat: 19, arabic: "سَبِّحِ ٱسْمَ رَبِّكَ ٱلْأَعْلَى", translation: "Yang Paling Tinggi" },
    { id: 30, name: "At-Tariq", ayat: 17, arabic: "وَٱلسَّمَآءِ وَٱلطَّارِقِ", translation: "Yang Datang di Malam Hari" },
];

// --- DAILY DHIKR DATA ---
const DAILY_DHIKR = {
    0: { day: 'Minggu', arabic: 'يَا حَيُّ يَا قَيُّومُ', translation: 'Wahai Yang Maha Hidup, Wahai Yang Berdiri Sendiri', benefit: 'Mendapat kemuliaan dan kemudahan dalam urusan.' },
    1: { day: 'Senin', arabic: 'لَا حَوْلَ وَلَا قُوَّةَ إِلَّا بِٱللَّهِ', translation: 'Tiada daya dan tiada kekuatan kecuali dengan pertolongan Allah', benefit: 'Dihindarkan dari kemudaratan dan dimudahkan rezeki.' },
    2: { day: 'Selasa', arabic: 'سُبْحَانَ ٱللَّهِ وَبِحَمْدِهِ', translation: 'Maha Suci Allah dengan segala puji bagi-Nya', benefit: 'Dihapuskan dosa-dosa walaupun sebanyak buih di lautan.' },
    3: { day: 'Rabu', arabic: 'أَسْتَغْفِرُ ٱللَّهَ ٱلْعَظِيمَ', translation: 'Aku memohon ampun kepada Allah Yang Maha Agung', benefit: 'Diampuni dosa dan mendapat ketenangan jiwa.' },
    4: { day: 'Kamis', arabic: 'سُبْحَانَ ٱللَّهِ ٱلْعَظِيمِ وَبِحَمْدِهِ', translation: 'Maha Suci Allah Yang Maha Agung dan dengan segala puji bagi-Nya', benefit: 'Ditanamkan satu pohon kurma di syurga.' },
    5: { day: 'Jumat', arabic: 'اللَّهُمَّ صَلِّ عَلَى مُحَمَّدٍ', translation: 'Ya Allah, berikan rahmat kepada Nabi Muhammad', benefit: 'Mendapat syafaat Nabi Muhammad SAW di Hari Kiamat.' },
    6: { day: 'Sabtu', arabic: 'لَا إِلَٰهَ إِلَّا ٱللَّهُ', translation: 'Tiada Tuhan selain Allah', benefit: 'Membuka pintu rezeki dan kebahagiaan.' },
};

// Function to get today's Dhikr
const getTodaysDhikr = () => {
    // getDay() returns 0 for Sunday, 1 for Monday, ..., 6 for Saturday
    const todayIndex = new Date().getDay(); 
    return DAILY_DHIKR[todayIndex];
};


// The main application component
export default function App() {
    // --- STATE MANAGEMENT ---
    // State to hold Firebase instances for use in saving functions
    const [db, setDb] = useState(null); 
    const [userId, setUserId] = useState(null);
    const [isLoading, setIsLoading] = useState(true);
    const [activeTab, setActiveTab] = useState('Home'); 
    const [prayerTimes, setPrayerTimes] = useState([]);
    const [prayerRecords, setPrayerRecords] = useState({});
    const [isTrackingLoading, setIsTrackingLoading] = useState(false);
    
    // NEW: Haid/Moon Tracker State
    const [haidData, setHaidData] = useState({});
    
    // --- DUMMY DATA ---
    const userName = “Fallenaru”;
    const location = “Malang, Indonesia";
    const today = new Date();
    const dateGregorian = today.toLocaleDateString('id-ID', { year: 'numeric', month: 'long', day: 'numeric', weekday: 'long' });
    const dateHijri = "11 Rajab 1447"; 
    
    const arabicFontClasses = "font-['Noto_Sans_Arabic'] text-right";
    
    // --- CALCULATED STATUSES ---
    const haidStatus = useMemo(() => getHaidStatus(haidData), [haidData]);
    const todaysDhikr = useMemo(() => getTodaysDhikr(), []); // Calculate Dhikr once per day

    const activePrayerData = useMemo(() => {
        if (prayerTimes.length === 0) return null;
        
        const nowMs = new Date().getTime();
        const sortedPrayers = [...prayerTimes]; 
        let activePrayer = null;
        let nextPrayer = null;

        for (let i = sortedPrayers.length - 1; i >= 0; i--) {
            const prayer = sortedPrayers[i];
            const prayerStartMs = getPrayerDate(prayer.time).getTime();
            
            if (nowMs >= prayerStartMs) {
                activePrayer = prayer;
                nextPrayer = sortedPrayers[i + 1];
                break;
            }
        }

        if (activePrayer) {
            if (!nextPrayer) {
                // If it's Isha, the next prayer is Fajr the next day, but we use the stored Fajr time
                nextPrayer = sortedPrayers.find(p => p.key === 'Fajr');
            }
            
            const startTime12hr = formatTime12h(activePrayer.time);
            const endTime12hr = formatTime12h(nextPrayer ? nextPrayer.time : '04:00'); 
            
            const startTimeDisplay = startTime12hr.replace(';', ':');
            const endTimeDisplay = endTime12hr.replace(';', ':');
            
            return {
                ...activePrayer,
                timeRange: `${startTimeDisplay} - ${endTimeDisplay}`,
                isDone: !!prayerRecords[activePrayer.key],
            };
        }
        
        // Before Fajr (Midnight to Fajr)
        const fajr = sortedPrayers.find(p => p.key === 'Fajr');
        const dhuhr = sortedPrayers.find(p => p.key === 'Dhuhr');
        if (fajr) {
            const startTime12hr = formatTime12h(fajr.time);
            const endTime12hr = formatTime12h(dhuhr ? dhuhr.time : '08:00'); 
            
            const startTimeDisplay = startTime12hr.replace(':', ';');
            const endTimeDisplay = endTime12hr.replace(':', ';');
            
            return {
                ...fajr,
                timeRange: `Menuju ${startTimeDisplay}`, 
                isDone: !!prayerRecords[fajr.key],
                isUpcoming: true,
                name: "Subuh (Berikutnya)"
            };
        }

        return null;
    }, [prayerTimes, prayerRecords]);
    
    // --- FIREBASE INITIALIZATION, AUTHENTICATION, AND LISTENERS (Refactored to one effect) ---
    useEffect(() => {
        if (Object.keys(firebaseConfig).length === 0) {
            console.error("[FIREBASE] Konfigurasi Firebase hilang.");
            setIsLoading(false);
            return () => {};
        }

        const app = initializeApp(firebaseConfig);
        const firestore = getFirestore(app);
        const firebaseAuth = getAuth(app);
        
        // Simpan instance firestore untuk fungsi penyimpanan (saveHaidData, trackPrayer)
        setDb(firestore); 

        let unsubRecords = () => {};
        let unsubHaid = () => {};

        const authenticate = async () => {
            // Guard against redundant sign-in attempts
            if (firebaseAuth.currentUser) return; 

            try {
                if (initialAuthToken) {
                    await signInWithCustomToken(firebaseAuth, initialAuthToken);
                    console.log("[FIREBASE] AUTH: Berhasil masuk dengan token kustom.");
                } else {
                    await signInAnonymously(firebaseAuth);
                    console.log("[FIREBASE] AUTH: Berhasil masuk secara anonim.");
                }
            } catch (error) {
                console.error("[FIREBASE] ERROR: Kesalahan saat masuk. Memastikan isLoading false.", error);
                setIsLoading(false);
            }
        };

        // Listener Perubahan Status Autentikasi
        const unsubscribeAuth = onAuthStateChanged(firebaseAuth, (user) => {
            // 1. Bersihkan listener lama
            unsubRecords();
            unsubHaid();

            if (user) {
                setUserId(user.uid);
                setIsLoading(false);
                console.log(`[FIREBASE] SUCCESS: Pengguna diautentikasi. UID: ${user.uid}. Memasang listener Firestore.`);

                // --- 2. PASANG FIRESTORE LISTENERS DI SINI ---
                const todayDocId = new Date().toISOString().slice(0, 10); 
                
                // Prayer Records Listener
                const userRecordsCollectionRef = collection(firestore, `artifacts/${appId}/users/${user.uid}/prayer_records`);
                const todayDocRef = doc(userRecordsCollectionRef, todayDocId);

                unsubRecords = onSnapshot(todayDocRef, (docSnapshot) => {
                    setPrayerRecords(docSnapshot.exists() ? docSnapshot.data() : {});
                    console.log("[FIRESTORE] LIVE: Menerima pembaruan catatan salat.");
                }, (error) => {
                    // Ini mungkin menunjukkan masalah koneksi persisten
                    console.error("[FIRESTORE] ERROR: Gagal mendengarkan catatan salat (periksa koneksi/aturan):", error);
                });

                // Haid Tracker Listener
                const haidDocRef = doc(firestore, `artifacts/${appId}/users/${user.uid}/settings/haid_tracker`);
                unsubHaid = onSnapshot(haidDocRef, (docSnapshot) => {
                    setHaidData(docSnapshot.exists() ? docSnapshot.data() : {});
                    console.log("[FIRESTORE] LIVE: Menerima pembaruan data haid.");
                }, (error) => {
                    console.error("[FIRESTORE] ERROR: Gagal mendengarkan pelacak haid (periksa koneksi/aturan):", error);
                });
                
            } else {
                setUserId(null); // Pastikan ID pengguna kosong
                console.log("[FIREBASE] AUTH: Pengguna tidak ditemukan. Mencoba masuk...");
                authenticate();
            }
        });

        // Fungsi pembersihan
        return () => {
            console.log("[FIREBASE] CLEANUP: Membatalkan langganan semua listener dan autentikasi.");
            unsubscribeAuth();
            unsubRecords();
            unsubHaid();
        };
    }, []); 
    
    // --- FIREBASE SAVERS ---
    
    const saveHaidData = async (data) => {
        // Menggunakan 'db' (firestore instance) yang disimpan di state
        if (!db || !userId) return; 
        const haidDocRef = doc(db, `artifacts/${appId}/users/${userId}/settings/haid_tracker`);
        try {
            await setDoc(haidDocRef, data, { merge: true });
            console.log("[FIRESTORE] SAVE: Data Haid berhasil disimpan.");
        } catch (error) {
            console.error("[FIRESTORE] ERROR: Kesalahan menyimpan data haid:", error);
        }
    };

    // Function to track a prayer as completed
    const trackPrayer = async (prayerKey) => {
        // Menggunakan 'db' (firestore instance) yang disimpan di state
        if (!db || !userId || isTrackingLoading || haidStatus.isHaid) return; 
        setIsTrackingLoading(true);

        const todayDocId = new Date().toISOString().slice(0, 10);
        const userRecordsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/prayer_records`);
        const todayDocRef = doc(userRecordsCollectionRef, todayDocId);

        try {
            await setDoc(todayDocRef, { 
                [prayerKey]: true, 
                updatedAt: new Date().toISOString() 
            }, { merge: true });
            console.log(`[FIRESTORE] SAVE: Salat ${prayerKey} berhasil dilacak.`);
        } catch (error) {
            console.error("[FIRESTORE] ERROR: Kesalahan melacak salat:", error);
        } finally {
            setIsTrackingLoading(false);
        }
    };


    // --- DHIKR & PRAYER TIMES (Dummy/API) ---

    // Function to fetch actual prayer times (Dummy)
    const fetchPrayerTimes = useCallback(() => {
        // Dummy times for Jakarta
        const fajrTime = '04:35';
        const dhuhrTime = '12:05';
        const asrTime = '15:15';
        const maghribTime = '18:10';
        const ishaTime = '19:25';

        const realPrayerTimes = [
            { name: "Subuh", time: fajrTime, key: 'Fajr' },
            { name: "Dzuhur", time: dhuhrTime, key: 'Dhuhr' },
            { name: "Ashar", time: asrTime, key: 'Asr' }, 
            { name: "Maghrib", time: maghribTime, key: 'Maghrib' },
            { name: "Isya", time: ishaTime, key: 'Isha' }
        ].sort((a, b) => {
            const timeA = a.time.replace(':', '');
            const timeB = b.time.replace(':', '');
            return timeA - timeB;
        });

        setPrayerTimes(realPrayerTimes);
    }, []);

    useEffect(() => {
        fetchPrayerTimes();
    }, [fetchPrayerTimes]);

    // --- APPLICATION COMPONENTS ---

    // Nav Item Component
    const NavItem = ({ children, isActive, label, onClick }) => (
        <button 
            className={`flex flex-col items-center p-1 transition-colors focus:outline-none active:scale-95 ${isActive ? 'text-teal-600' : 'text-gray-400 hover:text-gray-500'}`}
            aria-label={label}
            onClick={onClick}
        >
            <span className="w-6 h-6 mb-0.5">{children}</span>
            <div className="text-xs font-medium">{label}</div>
        </button>
    );

    // Active Prayer Card Component
    const ActivePrayerCard = ({ prayerData, haidStatus }) => {
        if (!prayerData) return (
            <div className="bg-amber-100 p-4 rounded-xl shadow-lg text-center">
                <p className="font-semibold text-amber-700">Memuat waktu salat...</p>
            </div>
        );

        let statusMessage = '';
        let buttonText = '';
        let buttonAction = () => {};
        let buttonDisabled = false;

        if (haidStatus.isHaid) {
            statusMessage = `Haid (H${haidStatus.haidDay}): Salat ditangguhkan.`;
            buttonText = 'Moon Mode Aktif';
            buttonDisabled = true;
        } else if (prayerData.isDone) {
            statusMessage = `Alhamdulillah! Salat ${prayerData.name} telah ditunaikan.`;
            buttonText = 'Selesai';
            buttonDisabled = true;
        } else {
            statusMessage = `Sekarang Waktu ${prayerData.name}. Segera tunaikan salat!`;
            buttonText = 'Tandai Selesai';
            buttonAction = () => trackPrayer(prayerData.key);
            buttonDisabled = isTrackingLoading;
        }

        return (
            <div className="relative bg-white rounded-xl overflow-hidden shadow-2xl transition-all hover:scale-[1.01] duration-300">
                <ThematicBackground />
                <div className="relative p-6 md:p-8 text-white z-10 space-y-4">
                    <p className="text-sm font-light uppercase tracking-widest opacity-80">Waktu Salat Saat Ini</p>
                    <h2 className="text-3xl md:text-5xl font-extrabold drop-shadow-lg">{prayerData.name}</h2>
                    <p className="text-lg md:text-xl font-medium flex items-center gap-2">
                        <svg className="w-5 h-5" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M10 2v3a2 2 0 0 0 2 2h4a2 2 0 0 0 2-2V2"/><path d="M4 22h16"/><path d="M4 11h16"/><path d="M12 11V2"/><path d="M12 22V11"/></svg>
                        {prayerData.time.replace(':', ';')} WIB
                    </p>
                    <p className="text-xs italic opacity-90">{prayerData.timeRange}</p>

                    <div className="pt-4 flex flex-col sm:flex-row items-start sm:items-center justify-between gap-3">
                        <p className={`text-sm md:text-lg font-bold ${haidStatus.isHaid ? 'text-pink-100' : 'text-green-100'}`}>
                            {statusMessage}
                        </p>
                        <button
                            onClick={buttonAction}
                            disabled={buttonDisabled}
                            className={`px-4 py-2 text-sm font-bold rounded-full transition-all duration-300 shadow-xl min-w-[150px] whitespace-nowrap ${
                                haidStatus.isHaid
                                    ? 'bg-pink-700 text-pink-100 cursor-not-allowed disabled:opacity-80'
                                    : prayerData.isDone
                                        ? 'bg-green-600 text-white cursor-default'
                                        : 'bg-white text-amber-700 hover:bg-gray-100 active:scale-95 disabled:opacity-50'
                            }`}
                        >
                            {buttonText}
                        </button>
                    </div>
                </div>
            </div>
        );
    };
    
    // Daily Dhikr Card Component
    const DailyDhikrCard = ({ dhikrData }) => {
        return (
            <div className="bg-white p-5 rounded-xl shadow-lg border-t-4 border-teal-500/50">
                <h3 className="text-xl font-extrabold text-teal-800 flex items-center mb-3">
                    <SparkleIcon className="w-5 h-5 mr-2 text-teal-600 fill-yellow-400"/> Dzikir Hari {dhikrData.day}
                </h3>
                
                <div className="bg-teal-50 p-4 rounded-lg">
                    <p className={`text-3xl ${arabicFontClasses} leading-relaxed text-teal-900 mb-2`}>
                        {dhikrData.arabic}
                    </p>
                    <p className="text-sm font-medium text-teal-700">
                        {dhikrData.translation}
                    </p>
                </div>
                
                <p className="mt-3 text-xs italic text-gray-500">
                    Keutamaan: {dhikrData.benefit}
                </p>
            </div>
        );
    };


    // Prayer List Item Component (Updated with Haid Status Check)
    const PrayerListItem = ({ prayer }) => {
        const isDone = !!prayerRecords[prayer.key];
        const hasPassed = isTimePassed(prayer.time);
        const isEarly = isWithinEarlyWindow(prayer.time);
        const isHaid = haidStatus.isHaid;

        let statusText = 'Akan Datang';
        let statusColor = 'text-gray-500';
        let buttonDisabled = isDone || isHaid;

        if (isHaid) {
            statusText = `Ditangguhkan (H${haidStatus.haidDay})`;
            statusColor = 'text-pink-600 font-bold';
            buttonDisabled = true;
        } else if (isDone) {
            statusText = 'Ditunaikan'; 
            statusColor = 'text-green-600 font-bold';
        } else if (hasPassed) {
            statusText = 'Terlewat';
            statusColor = 'text-red-500 font-bold';
            buttonDisabled = true; 
        } else if (isEarly) {
            statusText = 'Waktu Qabliyyah';
            statusColor = 'text-blue-500 font-bold';
            buttonDisabled = isTrackingLoading;
        } else {
            buttonDisabled = true; 
        }

        return (
            <div className="flex items-center justify-between p-4 bg-white rounded-xl shadow-md border-l-4 border-amber-500/20 mb-3 transition-shadow hover:shadow-lg">
                <div className="flex flex-col">
                    <span className="text-lg font-bold text-gray-800">{prayer.name}</span>
                    <span className="text-sm text-amber-600 font-semibold">{prayer.time.replace(':', ';')} WIB</span>
                </div>
                <div className="flex items-center gap-3">
                    <span className={`text-sm ${statusColor} min-w-[70px] text-right`}>
                        {isDone && !isHaid ? <CheckCircleIcon className="w-5 h-5 inline mr-1" /> : ''}
                        {statusText}
                    </span>
                    <button
                        onClick={() => trackPrayer(prayer.key)}
                        disabled={buttonDisabled}
                        className={`px-3 py-1 text-sm font-semibold rounded-full transition-all duration-300 shadow-md ${
                            isDone && !isHaid
                                ? 'bg-green-100 text-green-700 cursor-default shadow-inner' 
                                : isHaid
                                    ? 'bg-pink-100 text-pink-700 cursor-not-allowed disabled:opacity-70'
                                    : (isEarly || !hasPassed) 
                                        ? 'bg-amber-600 text-white hover:bg-amber-700 active:scale-98 disabled:opacity-50'
                                        : 'bg-gray-300 text-gray-600 cursor-not-allowed'
                        }`}
                    >
                        {isDone && !isHaid ? 'Selesai' : isHaid ? 'Moon Mode' : (isTrackingLoading ? 'Memproses...' : 'Tandai Selesai')}
                    </button>
                </div>
            </div>
        );
    };

    // Prayer Tracker Visualizer Component
    const PrayerTrackerVisualizer = ({ prayerTimes, prayerRecords, isHaid }) => {
        return (
            <div className="flex justify-between items-center space-x-2 w-full px-1">
                {prayerTimes.map((prayer) => {
                    const isDone = !!prayerRecords[prayer.key];
                    const hasPassed = isTimePassed(prayer.time);
                    
                    let bgColor = isHaid ? 'bg-pink-300' : 'bg-gray-300'; // Default: Not yet time
                    let tooltip = isHaid ? 'Moon Mode: Salat ditangguhkan.' : `Waktu ${prayer.name} belum tiba.`;

                    if (!isHaid) {
                        if (isDone) {
                            bgColor = 'bg-green-500'; // Green: Done
                            tooltip = `${prayer.name}: Ditunaikan.`;
                        } else if (hasPassed) {
                            bgColor = 'bg-red-500'; // Red: Passed but not done
                            tooltip = `${prayer.name}: Terlewat! Segera ganti.`;
                        }
                    }

                    // Height is adjusted to make the tracker subtle but visible
                    return (
                        <div 
                            key={prayer.key} 
                            className="relative flex flex-col items-center group flex-grow"
                        >
                            <div 
                                className={`w-1 h-12 rounded-full transition-all duration-300 ${bgColor} group-hover:h-16`}
                                title={tooltip}
                            ></div>
                            <span className="text-xs text-gray-600 mt-1 font-medium">{prayer.name}</span>
                        </div>
                    );
                })}
            </div>
        );
    };
    
    // Moon Settings Component - Used by MoonModeContent
    const MoonSettings = () => {
        const [start, setStart] = useState(haidData.startDate || '');
        const [end, setEnd] = useState(haidData.endDate || '');
        const [isSaving, setIsSaving] = useState(false);

        const handleSave = async () => {
            if (!start || !end) return;
            setIsSaving(true);
            await saveHaidData({ startDate: start, endDate: end });
            setIsSaving(false);
        };

        const handleClear = async () => {
            setIsSaving(true);
            await saveHaidData({ startDate: '', endDate: '' });
            setStart('');
            setEnd('');
            setIsSaving(false);
        };
        
        const todayString = new Date().toISOString().slice(0, 10);
        
        return (
            <div className="p-5 space-y-6 bg-white rounded-xl shadow-lg border-t-4 border-pink-500">
                <h2 className="text-2xl font-bold text-pink-600 flex items-center">
                    <MoonIcon className="w-6 h-6 mr-2 fill-pink-600"/> Moon Mode (Haid Tracker)
                </h2>
                <p className="text-gray-600 text-sm">
                    Fitur ini ditujukan untuk wanita Muslim. Masukkan rentang tanggal haid Anda, dan aplikasi akan secara otomatis menangguhkan semua fitur pelacakan salat untuk hari-hari tersebut.
                </p>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <div>
                        <label htmlFor="startDate" className="block text-sm font-medium text-gray-700 mb-1">Tanggal Mulai (H1)</label>
                        <input 
                            id="startDate"
                            type="date"
                            value={start}
                            onChange={(e) => setStart(e.target.value)}
                            // Max date ensures you can't start a period in the future, 
                            // but allow editing past ones. Using todayString as a soft suggestion.
                            className="w-full p-2 border border-pink-300 rounded-lg focus:ring-pink-500 focus:border-pink-500"
                        />
                    </div>
                    <div>
                        <label htmlFor="endDate" className="block text-sm font-medium text-gray-700 mb-1">Tanggal Berakhir (Hari Terakhir)</label>
                        <input 
                            id="endDate"
                            type="date"
                            value={end}
                            onChange={(e) => setEnd(e.target.value)}
                            min={start}
                            className="w-full p-2 border border-pink-300 rounded-lg focus:ring-pink-500 focus:border-pink-500"
                        />
                    </div>
                </div>

                <div className="flex gap-3">
                    <button
                        onClick={handleSave}
                        disabled={isSaving || !start || !end || new Date(start) > new Date(end)}
                        className="flex-1 px-4 py-2 text-white font-semibold bg-pink-500 rounded-lg shadow-md hover:bg-pink-600 transition-all disabled:opacity-50"
                    >
                        {isSaving ? 'Menyimpan...' : 'Simpan Periode Haid'}
                    </button>
                    <button
                        onClick={handleClear}
                        disabled={isSaving || (!start && !end)}
                        className="px-4 py-2 text-pink-500 font-semibold bg-pink-50 border border-pink-500 rounded-lg shadow-md hover:bg-pink-100 transition-all disabled:opacity-50"
                    >
                        Hapus Periode
                    </button>
                </div>

                {haidStatus.isHaid && (
                    <div className="bg-pink-50 border-l-4 border-pink-500 p-4 rounded-md">
                        <p className="font-semibold text-pink-700">Status Hari Ini:</p>
                        <p className="text-xl font-extrabold text-pink-900">
                            H{haidStatus.haidDay}
                            <span className="text-sm font-normal text-pink-700 ml-2">
                                (Hari ke-{haidStatus.haidDay} dari {haidStatus.totalDays} hari)
                            </span>
                        </p>
                    </div>
                )}
                
                {!haidStatus.isHaid && (haidData.endDate && new Date(haidData.endDate) < new Date(todayString)) && (
                     <div className="bg-green-50 border-l-4 border-green-500 p-4 rounded-md">
                        <p className="font-semibold text-green-700">Periode Haid Selesai.</p>
                        <p className="text-sm text-green-600">Anda telah suci. Selamat melanjutkan ibadah salat!</p>
                    </div>
                )}
            </div>
        );
    };
    
    // NEW: Qibla Compass Component
    const QiblaCompass = ({ haidStatus }) => {
        const [userLocation, setUserLocation] = useState(null); // {lat, lon}
        const [qiblaAngle, setQiblaAngle] = useState(null); // Angle from North (0-360)
        const [deviceHeading, setDeviceHeading] = useState(0); // Device heading (0-360)
        const [error, setError] = useState(null);

        // 1. Get User Location and Calculate Qibla
        useEffect(() => {
            if (!navigator.geolocation) {
                setError('Geolocation tidak didukung oleh browser Anda.');
                return;
            }

            const success = (position) => {
                const { latitude, longitude } = position.coords;
                setUserLocation({ lat: latitude, lon: longitude });
                
                const angle = calculateQiblaAngle(latitude, longitude);
                setQiblaAngle(angle);
                setError(null);
                console.log(`[Qibla] SUCCESS: Lokasi: ${latitude}, ${longitude}. Sudut Kiblat (dari Utara): ${angle.toFixed(2)}°`);
            };

            // FIX: Improved error handling logging
            const failure = (err) => {
                let errorMessage;
                
                switch(err.code) {
                    case 1: // PERMISSION_DENIED
                        errorMessage = 'Akses lokasi ditolak. Mohon izinkan akses lokasi di pengaturan browser Anda.';
                        break;
                    case 2: // POSITION_UNAVAILABLE
                        errorMessage = 'Informasi lokasi tidak tersedia. Coba periksa koneksi internet Anda.';
                        break;
                    case 3: // TIMEOUT
                        errorMessage = 'Permintaan lokasi melebihi batas waktu (5 detik). Coba lagi.';
                        break;
                    default:
                        // Fallback for generic or unknown errors
                        errorMessage = `Gagal mendapatkan lokasi. Kode Error: ${err.code || 'unknown'}.`;
                        break;
                }
                
                setError(errorMessage);
                // Logging the descriptive message instead of the potentially empty error object
                console.error(`[Qibla] Geolocation Error: ${errorMessage}`, { code: err.code, message: err.message });
            };

            navigator.geolocation.getCurrentPosition(success, failure, {
                enableHighAccuracy: true,
                timeout: 5000,
                maximumAge: 0
            });
        }, []);

        // 2. Get Device Orientation (for a true compass experience)
        useEffect(() => {
            if (window.DeviceOrientationEvent) {
                const handleDeviceOrientation = (event) => {
                    let alpha = event.alpha; // Azimuth (rotation around the Z axis, 0 is North)
                    
                    if (alpha !== null) {
                        // Standard alpha often increases counter-clockwise, compass is clockwise.
                        // We use (360 - alpha) as a common compensation.
                        setDeviceHeading(360 - alpha); 
                    }
                };
                
                window.addEventListener('deviceorientation', handleDeviceOrientation, true);
                return () => {
                    window.removeEventListener('deviceorientation', handleDeviceOrientation, true);
                };
            } else {
                console.log("[Qibla] DeviceOrientationEvent tidak didukung.");
            }
        }, []);

        
        const qiblaText = qiblaAngle !== null 
            ? `${qiblaAngle.toFixed(1)}° dari Utara Sejati` 
            : 'Menghitung...';

        const locationText = userLocation 
            ? `Lokasi Anda: ${userLocation.lat.toFixed(4)}, ${userLocation.lon.toFixed(4)}`
            : 'Mencari lokasi...';

        // UI Rendering
        return (
            <div className="p-4 space-y-6 max-w-xl mx-auto">
                <div className="bg-white p-6 rounded-xl shadow-lg border-t-4 border-amber-500">
                    <h2 className="text-2xl font-bold text-amber-700 flex items-center mb-4">
                        <CompassIcon className="w-6 h-6 mr-2 text-amber-600"/> Kompas Kiblat
                    </h2>
                    
                    {error && (
                        <div className="bg-red-100 text-red-700 p-3 rounded-lg mb-4">
                            <p className="font-semibold">Kesalahan Geolocation:</p>
                            <p className="text-sm">{error}</p>
                            <p className="text-xs mt-1">Kompas Kiblat memerlukan akses lokasi untuk berfungsi.</p>
                        </div>
                    )}
                    
                    {haidStatus.isHaid && (
                        <div className="bg-pink-100 text-pink-700 p-3 rounded-lg mb-4">
                            <p className="font-semibold">Moon Mode Aktif:</p>
                            <p className="text-sm">Kompas Kiblat tetap tersedia, namun pelacakan salat dinonaktifkan.</p>
                        </div>
                    )}

                    <div className="text-center space-y-3">
                        <p className="text-sm text-gray-500">{locationText}</p>
                        <p className="text-xl font-extrabold text-amber-800">{qiblaText}</p>

                        {/* Compass Visualization */}
                        <div className="relative w-full aspect-square max-w-xs mx-auto mt-8 mb-6">
                            
                            {/* Compass Face - Rotates by Device Heading */}
                            <div 
                                className="absolute inset-0 border-4 border-amber-300 rounded-full bg-amber-50 shadow-inner transition-transform duration-300"
                                style={{
                                    // Rotate the entire compass face to align North ('N') with True North
                                    transform: `rotate(${deviceHeading}deg)`,
                                }}
                            >
                                {/* Direction Markers (N, E, S, W) - Fixed relative to the rotating face */}
                                <div className="absolute top-0 left-1/2 -translate-x-1/2 pt-1 font-bold text-lg text-red-600">N</div>
                                <div className="absolute bottom-0 left-1/2 -translate-x-1/2 pb-1 font-bold text-lg text-gray-700">S</div>
                                <div className="absolute left-0 top-1/2 -translate-y-1/2 pl-1 font-bold text-lg text-gray-700">W</div>
                                <div className="absolute right-0 top-1/2 -translate-y-1/2 pr-1 font-bold text-lg text-gray-700">E</div>
                                
                                {/* North Needle (Always points to 'N' on the face) */}
                                <div className="absolute top-0 left-1/2 -translate-x-1/2 w-0.5 h-[50%] bg-red-600 transform origin-bottom pt-1"></div>
                            </div>


                            {/* Qibla Indicator - Fixed relative to the device/screen */}
                            {qiblaAngle !== null && (
                                <div 
                                    className="absolute inset-0 transform transition-transform duration-500"
                                    style={{
                                        // Rotate the Qibla Indicator to the Qibla Angle, relative to True North (0deg)
                                        transform: `rotate(${qiblaAngle}deg)`,
                                    }}
                                >
                                    {/* Qibla Marker (Kaaba Icon) */}
                                    <div 
                                        className="absolute top-0 left-1/2 -translate-x-1/2 w-8 h-8 transform"
                                        style={{
                                            // Translate it outwards and counteract the rotation to keep the icon upright
                                            transform: `translate(0, -100%) rotate(${qiblaAngle * -1}deg)`,
                                            // The Kaaba icon is placed at the edge corresponding to the Qibla angle.
                                            marginTop: '8px' 
                                        }}
                                    >
                                        <LocationIcon className="w-8 h-8 text-black fill-yellow-600 shadow-md"/>
                                    </div>
                                    
                                    {/* Line pointing to Qibla */}
                                    <div className="absolute top-0 left-1/2 -translate-x-1/2 w-0.5 h-[50%] bg-green-500 transform origin-bottom pt-1 opacity-70"></div>
                                </div>
                            )}

                            {/* Center Dot */}
                            <div className="absolute top-1/2 left-1/2 w-4 h-4 -translate-x-1/2 -translate-y-1/2 rounded-full bg-amber-900 border-2 border-white shadow-xl z-10"></div>
                        </div>
                        
                        {/* Instructions / Current Device Heading */}
                        <div className="mt-4 pt-4 border-t border-gray-100">
                             {window.DeviceOrientationEvent ? (
                                <p className="text-xs text-gray-500 italic">
                                    Putar perangkat Anda untuk mengarahkan 'N' (merah) ke Utara. Ikon Ka'bah (kuning) menunjukkan arah Kiblat.
                                    <br/> 
                                    <span className='font-mono text-xs text-gray-700 font-semibold'>
                                        (Heading Ponsel: {deviceHeading.toFixed(1)}°)
                                    </span>
                                </p>
                            ) : (
                                <p className="text-xs text-gray-500 italic">
                                    Arah Kiblat (ikon Ka'bah) adalah {qiblaAngle !== null ? qiblaAngle.toFixed(1) : '...'}° dari Utara Sejati. Gunakan kompas fisik Anda untuk menentukan Kiblat.
                                </p>
                            )}
                        </div>
                    </div>
                </div>
            </div>
        );
    };

    // Quran Sub-Components
    const SurahDetail = ({ surah, setSelectedSurah }) => (
        <div className="p-6 bg-white rounded-xl shadow-lg">
            <button 
                onClick={() => setSelectedSurah(null)}
                className="mb-4 text-teal-600 font-medium hover:underline flex items-center"
            >
                &larr; Kembali ke Daftar Surah
            </button>
            
            <h1 className="text-3xl font-extrabold text-teal-800 border-b pb-2 mb-4">
                Surah {surah.name}
            </h1>
            <p className="text-xl italic text-gray-600 mb-6">"{surah.translation}" ({surah.ayat} Ayat)</p>

            <div className="space-y-6">
                {/* Arabic Text Display */}
                <div className="bg-teal-50 p-4 rounded-lg border-l-4 border-teal-500">
                    <p className={`text-4xl ${arabicFontClasses} leading-relaxed text-teal-900`}>
                        {surah.arabic}
                    </p>
                    <p className="mt-2 text-sm text-teal-700 font-semibold">
                        (Contoh Ayat Pertama)
                    </p>
                </div>

                {/* Example Translation */}
                <div className="text-gray-700">
                    <h3 className="text-lg font-bold mb-2">Terjemahan Ringkas:</h3>
                    <p className="italic border-l-2 pl-3 border-gray-300">
                        Surah ini {surah.translation.toLowerCase()} dimulai dengan {surah.arabic.split(' ')[0]} dan merupakan salah satu surah yang paling sering dibaca karena keutamaannya. Surah ini sering digunakan dalam shalat sehari-hari.
                    </p>
                </div>
            </div>
        </div>
    );

    const SurahList = ({ setSelectedSurah }) => (
        <div className="space-y-3">
            <h2 className="text-2xl font-bold text-gray-800">30 Surah Pendek Pilihan</h2> {/* Diperbarui dari 20 */}
            <p className="text-sm text-gray-500">Pilih surah untuk melihat teks Arab dan terjemahan (Semua data ini bersifat contoh dan disederhanakan).</p>
            <div className="grid grid-cols-1 sm:grid-cols-2 gap-3 pt-2">
                {SHORT_SURAHS.map(surah => (
                    <button
                        key={surah.id}
                        onClick={() => setSelectedSurah(surah)}
                        className="p-4 bg-white rounded-xl shadow-md border-l-4 border-teal-500/50 hover:bg-teal-50 transition-colors flex justify-between items-center"
                    >
                        <div>
                            <p className="text-lg font-semibold text-teal-700">Surah {surah.name}</p>
                            <p className="text-xs text-gray-500">{surah.translation} ({surah.ayat} Ayat)</p>
                        </div>
                        <span className={`text-2xl ${arabicFontClasses} text-teal-800`}>
                            {surah.arabic.split(' ')[0]}...
                        </span>
                    </button>
                ))}
            </div>
        </div>
    );
    
    // Quran View Component
    const QuranView = () => {
        const [selectedSurah, setSelectedSurah] = useState(null);
        
        return (
            <div className="p-4 max-w-4xl mx-auto">
                {selectedSurah ? <SurahDetail surah={selectedSurah} setSelectedSurah={setSelectedSurah} /> : <SurahList setSelectedSurah={setSelectedSurah} />}
            </div>
        );
    };
    
    // Moon Mode Content Component
    const MoonModeContent = () => (
        <div className="p-4 space-y-6 max-w-4xl mx-auto">
            <MoonSettings />
        </div>
    );


    // Home Content Component
    const HomeContent = () => (
        <div className="p-4 space-y-6 max-w-4xl mx-auto">
            
            {/* Active Prayer Card */}
            <ActivePrayerCard prayerData={activePrayerData} haidStatus={haidStatus} trackPrayer={trackPrayer}/>
            
            {/* Daily Dhikr Card */}
            <DailyDhikrCard dhikrData={todaysDhikr} />
            
            {/* Haid Status Banner */}
            {haidStatus.isHaid && (
                <div className="bg-pink-100 border-l-4 border-pink-500 p-4 rounded-xl shadow-md flex items-center justify-between">
                    <div className="flex items-center">
                        <MoonIcon className="w-6 h-6 text-pink-600 mr-3 fill-pink-600"/>
                        <div>
                            <p className="font-bold text-pink-800">Moon Mode Aktif!</p>
                            <p className="text-sm text-pink-700">Hari ini H{haidStatus.haidDay} dari {haidStatus.totalDays} hari. Salat ditangguhkan otomatis.</p>
                        </div>
                    </div>
                    <button 
                        onClick={() => setActiveTab('Moon')} // NAVIGASI KE TAB MOON
                        className="text-pink-600 text-sm font-semibold hover:underline bg-pink-50 px-3 py-1 rounded-full"
                    >
                        Kelola Haid &rarr;
                    </button>
                </div>
            )}

            {/* Prayer Tracker Visualizer */}
            <div className="bg-white p-4 rounded-xl shadow-lg">
                <h3 className="text-lg font-semibold text-gray-800 mb-4">Visualisasi Salat Hari Ini</h3>
                <PrayerTrackerVisualizer prayerTimes={prayerTimes} prayerRecords={prayerRecords} isHaid={haidStatus.isHaid} />
            </div>
            
            {/* Full Prayer List */}
            <div className="space-y-3 pt-4">
                <h3 className="text-xl font-bold text-gray-800">Jadwal Salat Harian</h3>
                {prayerTimes.map(p => <PrayerListItem key={p.key} prayer={p} />)}
            </div>
        </div>
    );
    
    // Settings Content Component (Simplified for General Settings)
    const SettingsContent = () => (
        <div className="p-4 space-y-6 max-w-4xl mx-auto">
            {/* General Settings Placeholder */}
            <div className="p-5 space-y-4 bg-white rounded-xl shadow-lg border-t-4 border-teal-500">
                <h2 className="text-2xl font-bold text-teal-600 flex items-center">
                    <SettingsIcon className="w-6 h-6 mr-2"/> Pengaturan Umum
                </h2>
                <div className="bg-gray-50 p-3 rounded-lg">
                    <p className="font-semibold text-gray-800">Tanggal Hari Ini:</p>
                    <p className="text-sm text-gray-600">{dateGregorian} ({dateHijri} H)</p>
                </div>
                <div className="bg-gray-50 p-3 rounded-lg flex items-center">
                    <LocationIcon className="w-4 h-4 text-gray-600 mr-2"/>
                    <p className="font-semibold text-gray-800">Lokasi Salat:</p>
                    <p className="text-sm text-gray-600 ml-2">{location}</p>
                </div>
                <div className="bg-gray-50 p-3 rounded-lg">
                    <p className="font-semibold text-gray-800">Akun Pengguna:</p>
                    <p className="text-sm text-gray-600">ID: {userId || 'Authenticating...'}</p>
                    <p className="text-sm text-gray-600">Nama: {userName}</p>
                </div>
            </div>
        </div>
    );

    // --- MAIN RENDER ---

    if (isLoading) {
        return (
            <div className="flex items-center justify-center min-h-screen bg-gray-50">
                <div className="animate-spin rounded-full h-10 w-10 border-b-2 border-teal-600"></div>
                <p className="ml-3 text-lg font-medium text-teal-700">Memuat Data...</p>
            </div>
        );
    }
    
    let content;
    switch (activeTab) {
        case 'Home':
            content = <HomeContent />;
            break;
        case 'Quran':
            content = <QuranView />;
            break;
        case 'Moon': 
            content = <MoonModeContent />;
            break;
        case 'Qibla': // NEW Case
            content = <QiblaCompass haidStatus={haidStatus} />;
            break;
        case 'Settings':
            content = <SettingsContent />;
            break;
        default:
            content = <HomeContent />;
    }

    return (
        <div className="min-h-screen bg-gray-50 flex flex-col font-[Inter]">
            {/* Header / Top Bar (Judul Diubah ke Stetsa Recap) */}
            <header className="p-4 bg-white shadow-md sticky top-0 z-20">
                <div className="flex justify-between items-center max-w-4xl mx-auto">
                    <div>
                        <h1 className="text-xl font-extrabold text-teal-700">Stetsa Recap</h1>
                        <p className="text-xs text-gray-500">{dateGregorian}</p>
                    </div>
                    {/* Placeholder for Profile/Notifications */}
                    <div className="flex items-center space-x-2">
                         <div className={`p-2 rounded-full ${haidStatus.isHaid ? 'bg-pink-100 border border-pink-500' : 'bg-green-100 border border-green-500'}`} title={haidStatus.isHaid ? `Moon Mode Aktif (H${haidStatus.haidDay})` : 'Mode Normal'}>
                            <MoonIcon className={`w-5 h-5 ${haidStatus.isHaid ? 'text-pink-600 fill-pink-600' : 'text-green-600'}`}/>
                        </div>
                    </div>
                </div>
            </header>

            {/* Main Content Area */}
            <main className="flex-grow pb-24">
                {content}
            </main>

            {/* Bottom Navigation */}
            <nav className="fixed bottom-0 left-0 right-0 bg-white border-t shadow-lg z-30">
                <div className="flex justify-around max-w-lg mx-auto py-2"> {/* Ukuran Navigasi Diperluas */}
                    <NavItem label="Beranda" isActive={activeTab === 'Home'} onClick={() => setActiveTab('Home')}>
                        <HomeIcon className={activeTab === 'Home' ? 'text-teal-600' : 'text-gray-400'}/>
                    </NavItem>
                    <NavItem label="Al-Qur'an" isActive={activeTab === 'Quran'} onClick={() => setActiveTab('Quran')}>
                        <QuranIcon className={activeTab === 'Quran' ? 'text-teal-600' : 'text-gray-400'}/>
                    </NavItem>
                     <NavItem label="Kiblat" isActive={activeTab === 'Qibla'} onClick={() => setActiveTab('Qibla')}>
                        <CompassIcon className={activeTab === 'Qibla' ? 'text-teal-600' : 'text-gray-400'}/>
                    </NavItem>
                    <NavItem label="Moon Mode" isActive={activeTab === 'Moon'} onClick={() => setActiveTab('Moon')}>
                        <MoonIcon className={activeTab === 'Moon' ? 'text-teal-600 fill-teal-600' : 'text-gray-400'}/>
                    </NavItem>
                    <NavItem label="Pengaturan" isActive={activeTab === 'Settings'} onClick={() => setActiveTab('Settings')}>
                        <SettingsIcon className={activeTab === 'Settings' ? 'text-teal-600' : 'text-gray-400'}/>
                    </NavItem>
                </div>
            </nav>
        </div>
    );
}
