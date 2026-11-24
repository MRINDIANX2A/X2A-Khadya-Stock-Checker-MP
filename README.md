<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
   <title>X2A MP</title>
    <!-- Firebase SDK (Cloud Database for Persistent Stock Updates) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, onSnapshot } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'x2a-khadya-stock-mp';
        let db = null;
        let auth = null;
        let isCloudReady = false;
        let unsubscribeCenters = null;

        function getStockDocRef() {
            if (!db || !auth?.currentUser) return null;
            return doc(db, 'artifacts', appId, 'public', 'data', 'mp_khadya_centers', 'main_stock');
        }

        function sanitizeCentersForFirestore(centersList) {
            if (!Array.isArray(centersList)) return [];
            const plain = JSON.parse(JSON.stringify(centersList));
            return plain
                .filter(c => c && typeof c === 'object')
                .map(c => ({
                    id: String(c.id || ('mp_center_' + Math.random().toString(36).slice(2, 8))),
                    name: String(c.name || ''),
                    district: String(c.district || ''),
                    districtHi: String(c.districtHi || c.district || ''),
                    tehsil: String(c.tehsil || ''),
                    type: String(c.type || 'सहकारी समिति केंद्र'),
                    address: String(c.address || ''),
                    phone: String(c.phone || ''),
                    timings: String(c.timings || 'सोम - शनि, सुबह 10:00 से शाम 05:00'),
                    stock: {
                        urea: Math.max(0, Number(c.stock?.urea) || 0),
                        dap: Math.max(0, Number(c.stock?.dap) || 0),
                        npk: Math.max(0, Number(c.stock?.npk) || 0),
                        ssp: Math.max(0, Number(c.stock?.ssp) || 0)
                    },
                    remarks: String(c.remarks || ''),
                    lastUpdated: String(c.lastUpdated || 'आज')
                }));
        }

        async function initCloudSync() {
            try {
                if (typeof __firebase_config === 'undefined' || !__firebase_config) {
                    return;
                }
                const firebaseConfig = JSON.parse(__firebase_config);
                const app = initializeApp(firebaseConfig);
                auth = getAuth(app);
                db = getFirestore(app);

                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }

                isCloudReady = true;
                const stockRef = getStockDocRef();
                if (!stockRef) return;

                if (unsubscribeCenters) {
                    unsubscribeCenters();
                }

                unsubscribeCenters = onSnapshot(stockRef, async (snap) => {
                    if (!auth.currentUser) return;
                    if (snap.exists()) {
                        const cloudData = snap.data();
                        if (Array.isArray(cloudData.centers) && cloudData.centers.length > 0) {
                            window.centersData = cloudData.centers;
                            try {
                                localStorage.setItem('x2a_khadya_stock_mp_v3', JSON.stringify(cloudData.centers));
                            } catch (e) {}
                            if (typeof window.updateCloudStatusUI === 'function') window.updateCloudStatusUI(true);
                            if (typeof window.updateDashboardStats === 'function') window.updateDashboardStats();
                            if (typeof window.applyFilters === 'function') window.applyFilters();
                        }
                    } else {
                        let initialList = window.DEFAULT_MP_CENTERS;
                        try {
                            const localRaw = localStorage.getItem('x2a_khadya_stock_mp_v3');
                            if (localRaw) {
                                const parsedLocal = JSON.parse(localRaw);
                                if (Array.isArray(parsedLocal) && parsedLocal.length > 0) {
                                    initialList = parsedLocal;
                                }
                            } else if (Array.isArray(window.centersData) && window.centersData.length > 0) {
                                initialList = window.centersData;
                            }
                        } catch (e) {}
                        await setDoc(stockRef, {
                            centers: sanitizeCentersForFirestore(initialList),
                            updatedAt: new Date().toISOString()
                        });
                        if (typeof window.updateCloudStatusUI === 'function') window.updateCloudStatusUI(true);
                    }
                }, (err) => {
                    console.error('Cloud stock listener error:', err);
                    if (typeof window.updateCloudStatusUI === 'function') window.updateCloudStatusUI(false);
                });
            } catch (err) {
                console.error('Failed to initialize cloud sync:', err);
                if (typeof window.updateCloudStatusUI === 'function') window.updateCloudStatusUI(false);
            }
        }

        window.persistCentersToCloud = async function(updatedCenters) {
            try {
                if (!isCloudReady || !auth?.currentUser) {
                    await initCloudSync();
                }
                const stockRef = getStockDocRef();
                if (!stockRef) return false;
                await setDoc(stockRef, {
                    centers: sanitizeCentersForFirestore(updatedCenters),
                    updatedAt: new Date().toISOString()
                }, { merge: true });
                return true;
            } catch (err) {
                console.error('Error saving centers to cloud:', err);
                return false;
            }
        };

        initCloudSync();
    </script>
    <!-- Google Fonts: Inter & Noto Sans Devanagari -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=Noto+Sans+Devanagari:wght@400;500;600;700;800&display=swap" rel="stylesheet">
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['"Noto Sans Devanagari"', 'Inter', 'system-ui', 'sans-serif'],
                    },
                    colors: {
                        agri: {
                            50: '#f0fdf4',
                            100: '#dcfce7',
                            200: '#bbf7d0',
                            500: '#22c55e',
                            600: '#16a34a',
                            700: '#15803d',
                            800: '#166534',
                            900: '#14532d',
                        },
                        saffron: {
                            500: '#ff9933',
                            600: '#ea580c',
                        }
                    }
                }
            }
        }
    </script>
    <style>
        @keyframes marqueeScroll {
            0% { transform: translateX(0%); }
            100% { transform: translateX(-50%); }
        }
        .animate-marquee {
            display: flex;
            width: max-content;
            animation: marqueeScroll 32s linear infinite;
        }
        .animate-marquee:hover {
            animation-play-state: paused;
        }

        body.sunlight-mode {
            background-color: #ffffff !important;
            color: #000000 !important;
        }
        body.sunlight-mode .bg-slate-50,
        body.sunlight-mode .bg-slate-100 {
            background-color: #f8fafc !important;
        }
        body.sunlight-mode article,
        body.sunlight-mode section,
        body.sunlight-mode header {
            border-color: #0f172a !important;
            border-width: 2px !important;
            box-shadow: none !important;
        }
        body.sunlight-mode p,
        body.sunlight-mode span,
        body.sunlight-mode label {
            color: #0f172a !important;
        }

        @media print {
            body * {
                visibility: hidden;
            }
            #printableTokenSlip, #printableTokenSlip * {
                visibility: visible;
            }
            #printableTokenSlip {
                position: fixed;
                left: 0;
                top: 0;
                width: 100%;
                border: 2px solid #000 !important;
                box-shadow: none !important;
            }
        }
    </style>
</head>
<body class="bg-slate-50 text-slate-800 antialiased min-h-screen flex flex-col pb-20 md:pb-0 selection:bg-agri-200 selection:text-agri-900">

    <!-- Top Tricolor Accent Bar -->
    <div class="h-1.5 w-full bg-gradient-to-r from-[#FF9933] via-white to-[#138808] sticky top-0 z-50 shadow-sm"></div>

    <!-- Navigation Header -->
    <header class="bg-white border-b border-slate-200 sticky top-1.5 z-40 shadow-sm">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <div class="flex flex-col lg:flex-row lg:items-center lg:justify-between py-3 gap-3">
                <!-- Brand Logo & Name -->
                <div class="flex items-center justify-between">
                    <div class="flex items-center gap-3">
                        <div class="w-11 h-11 rounded-xl bg-gradient-to-br from-agri-600 to-agri-800 flex items-center justify-center text-white shadow-md shadow-agri-600/20 shrink-0">
                            <i data-lucide="sprout" class="w-6 h-6"></i>
                        </div>
                        <div>
                            <div class="flex items-center gap-2">
                                <h1 class="text-lg sm:text-2xl font-extrabold tracking-tight text-slate-900">
                                    X2A <span class="text-agri-700">Khadya</span> Stock Checker
                                </h1>
                                <span id="cloudSyncBadge" class="inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-semibold bg-agri-100 text-agri-800 border border-agri-200">
                                    <span id="cloudSyncDot" class="w-1.5 h-1.5 rounded-full bg-agri-600 animate-pulse"></span>
                                    <span id="cloudSyncText">MP LIVE</span>
                                </span>
                            </div>
                            <p class="text-xs sm:text-sm text-slate-500 font-medium">
                                मध्य प्रदेश खाद स्टॉक ट्रैकर • <span class="text-agri-700 font-semibold">संचालक: MR.INDIAN X2A (YT CHANNEL)</span>
                            </p>
                        </div>
                    </div>
                </div>

                <!-- Right Header Actions -->
                <div class="flex flex-wrap items-center gap-2 self-start lg:self-auto">
                    <button onclick="openCalculatorModal()" class="inline-flex items-center gap-1.5 px-3 py-2 rounded-xl text-xs sm:text-sm font-bold text-agri-800 bg-agri-50 hover:bg-agri-100 border border-agri-200 transition-colors cursor-pointer" title="खाद कैलकुलेटर व सरकारी रेट">
                        <i data-lucide="calculator" class="w-4 h-4 text-agri-700"></i>
                        <span>रेट व कैलकुलेटर</span>
                    </button>
                    <button id="sunlightToggleBtn" onclick="toggleSunlightMode()" class="inline-flex items-center gap-1.5 px-3 py-2 rounded-xl text-xs sm:text-sm font-semibold text-slate-700 bg-slate-100 hover:bg-slate-200 transition-colors cursor-pointer" title="तेज धूप में साफ देखने के लिए हाई-कंट्रास्ट मोड">
                        <i data-lucide="sun" class="w-4 h-4 text-amber-500"></i>
                        <span class="hidden sm:inline">धूप मोड</span>
                    </button>
                    <button onclick="openAdminModal()" class="inline-flex items-center gap-1.5 px-3.5 py-2 rounded-xl text-xs sm:text-sm font-bold text-white bg-slate-900 hover:bg-slate-800 active:scale-[0.98] transition-all shadow-sm cursor-pointer">
                        <i data-lucide="shield-check" class="w-4 h-4 text-saffron-500"></i>
                        <span>Admin Update</span>
                    </button>
                </div>
            </div>
        </div>
    </header>

    <!-- Main Content Container -->
    <main class="flex-1 max-w-7xl w-full mx-auto px-4 sm:px-6 lg:px-8 py-6 space-y-6">

        <!-- Summary Statistics Cards -->
        <section aria-label="स्टॉक सारांश" class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
            <div class="bg-white rounded-2xl p-4 sm:p-5 border border-slate-200/90 shadow-sm flex items-center justify-between">
                <div>
                    <p class="text-xs font-semibold uppercase tracking-wider text-slate-400">कुल समितियाँ (MP)</p>
                    <p id="statTotalCenters" class="text-2xl sm:text-3xl font-extrabold text-slate-900 mt-1">0</p>
                    <p class="text-xs text-slate-500 mt-0.5">मध्य प्रदेश वितरण केंद्र</p>
                </div>
                <div class="w-12 h-12 rounded-2xl bg-slate-100 text-slate-700 flex items-center justify-center shrink-0">
                    <i data-lucide="warehouse" class="w-6 h-6"></i>
                </div>
            </div>

            <div class="bg-white rounded-2xl p-4 sm:p-5 border border-slate-200/90 shadow-sm flex items-center justify-between">
                <div>
                    <p class="text-xs font-semibold uppercase tracking-wider text-blue-600">कुल यूरिया (Urea)</p>
                    <p id="statTotalUrea" class="text-2xl sm:text-3xl font-extrabold text-slate-900 mt-1">0</p>
                    <p class="text-xs text-slate-500 mt-0.5">उपलब्ध कट्टे (Bags)</p>
                </div>
                <div class="w-12 h-12 rounded-2xl bg-blue-50 text-blue-600 flex items-center justify-center shrink-0">
                    <i data-lucide="package" class="w-6 h-6"></i>
                </div>
            </div>

            <div class="bg-white rounded-2xl p-4 sm:p-5 border border-slate-200/90 shadow-sm flex items-center justify-between">
                <div>
                    <p class="text-xs font-semibold uppercase tracking-wider text-emerald-600">कुल डीएपी (DAP)</p>
                    <p id="statTotalDap" class="text-2xl sm:text-3xl font-extrabold text-slate-900 mt-1">0</p>
                    <p class="text-xs text-slate-500 mt-0.5">उपलब्ध कट्टे (Bags)</p>
                </div>
                <div class="w-12 h-12 rounded-2xl bg-emerald-50 text-emerald-600 flex items-center justify-center shrink-0">
                    <i data-lucide="layers" class="w-6 h-6"></i>
                </div>
            </div>

            <div class="bg-white rounded-2xl p-4 sm:p-5 border border-slate-200/90 shadow-sm flex items-center justify-between">
                <div>
                    <p class="text-xs font-semibold uppercase tracking-wider text-amber-600">कम स्टॉक / खाली</p>
                    <p id="statLowOrOut" class="text-2xl sm:text-3xl font-extrabold text-slate-900 mt-1">0</p>
                    <p class="text-xs text-slate-500 mt-0.5">ध्यान देने योग्य केंद्र</p>
                </div>
                <div class="w-12 h-12 rounded-2xl bg-amber-50 text-amber-600 flex items-center justify-center shrink-0">
                    <i data-lucide="alert-triangle" class="w-6 h-6"></i>
                </div>
            </div>
        </section>

        <!-- Search & Filter Control Panel -->
        <section class="bg-white rounded-2xl p-4 sm:p-5 border border-slate-200 shadow-sm space-y-4">
            <div class="flex flex-wrap items-center justify-between gap-2 border-b border-slate-100 pb-3">
                <div class="flex items-center gap-2">
                    <i data-lucide="filter" class="w-4 h-4 text-agri-600"></i>
                    <h3 class="font-bold text-slate-800 text-sm sm:text-base">मध्य प्रदेश सहकारी समिति खोजें (MP Search & Filter)</h3>
                </div>
                <div class="flex items-center gap-2">
                    <button id="favFilterToggleBtn" onclick="toggleFavoritesFilter()" class="inline-flex items-center gap-1.5 text-xs font-bold px-3 py-1 rounded-lg border border-amber-300 bg-amber-50 text-amber-800 hover:bg-amber-100 transition-colors cursor-pointer">
                        <i data-lucide="star" class="w-3.5 h-3.5 fill-amber-400 text-amber-500"></i>
                        <span>पसंदीदा (<span id="favCountBadge">0</span>)</span>
                    </button>
                    <button onclick="resetFilters()" class="inline-flex items-center gap-1 text-xs font-semibold px-2.5 py-1 rounded-lg bg-slate-100 hover:bg-slate-200 text-slate-600 cursor-pointer" title="फ़िल्टर रीसेट करें">
                        <i data-lucide="rotate-ccw" class="w-3.5 h-3.5"></i> रीसेट
                    </button>
                    <span id="resultCountBadge" class="text-xs font-semibold px-2.5 py-1 rounded-lg bg-agri-50 text-agri-800 border border-agri-200">
                        0 समितियाँ दिख रही हैं
                    </span>
                </div>
            </div>

            <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-3.5">
                <!-- Search Input with Voice Search Mic -->
                <div class="relative">
                    <label for="searchInput" class="block text-xs font-semibold text-slate-500 mb-1">समिति, गाँव या जिला खोजें (बोलकर या लिखकर)</label>
                    <div class="relative flex items-center">
                        <i data-lucide="search" class="w-4 h-4 text-slate-400 absolute left-3.5"></i>
                        <input type="text" id="searchInput" oninput="applyFilters()" placeholder="जैसे: चटुखेड़ा, इंदौर, ब्यावरा..."
                            class="w-full pl-10 pr-11 py-2.5 bg-slate-50 border border-slate-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-agri-500 focus:bg-white transition-all">
                        <button type="button" id="voiceSearchBtn" onclick="startVoiceSearch()" title="बोलकर खोजें (Voice Search)"
                            class="absolute right-2 w-8 h-8 rounded-lg bg-agri-100 hover:bg-agri-200 text-agri-700 flex items-center justify-center transition-colors cursor-pointer">
                            <i data-lucide="mic" class="w-4 h-4"></i>
                        </button>
                    </div>
                </div>

                <!-- District Filter -->
                <div>
                    <label for="districtFilter" class="block text-xs font-semibold text-slate-500 mb-1">जिला चुनें (MP District)</label>
                    <select id="districtFilter" onchange="applyFilters()"
                        class="w-full px-3.5 py-2.5 bg-slate-50 border border-slate-200 rounded-xl text-sm font-medium text-slate-700 focus:outline-none focus:ring-2 focus:ring-agri-500 focus:bg-white transition-all cursor-pointer">
                        <option value="ALL">सभी जिले (All MP Districts)</option>
                        <option value="Rajgarh">राजगढ़ (Rajgarh) - चटुखेड़ा</option>
                        <option value="Indore">इंदौर (Indore)</option>
                        <option value="Ujjain">उज्जैन (Ujjain)</option>
                        <option value="Bhopal">भोपाल (Bhopal)</option>
                        <option value="Sehore">सीहोर (Sehore)</option>
                        <option value="Dewas">देवास (Dewas)</option>
                        <option value="Shajapur">शाजापुर (Shajapur)</option>
                        <option value="Agar Malwa">आगर मालवा (Agar Malwa)</option>
                        <option value="Vidisha">विदिशा (Vidisha)</option>
                        <option value="Raisen">रायसेन (Raisen)</option>
                        <option value="Mandsaur">मंदसौर (Mandsaur)</option>
                        <option value="Neemuch">नीमच (Neemuch)</option>
                        <option value="Ratlam">रतलाम (Ratlam)</option>
                        <option value="Gwalior">ग्वालियर (Gwalior)</option>
                        <option value="Morena">मुरैना (Morena)</option>
                        <option value="Shivpuri">शिवपुरी (Shivpuri)</option>
                        <option value="Guna">गुना (Guna)</option>
                        <option value="Sagar">सागर (Sagar)</option>
                        <option value="Damoh">दमोह (Damoh)</option>
                        <option value="Chhatarpur">छतरपुर (Chhatarpur)</option>
                        <option value="Tikamgarh">टीकमगढ़ / निवाड़ी</option>
                        <option value="Jabalpur">जबलपुर (Jabalpur)</option>
                        <option value="Chhindwara">छिंदवाड़ा / पांढुर्णा</option>
                        <option value="Seoni">सिवनी (Seoni)</option>
                        <option value="Narmadapuram">नर्मदापुरम (Hoshangabad)</option>
                        <option value="Rewa">रीवा / मऊगंज</option>
                        <option value="Satna">सतना / मैहर</option>
                        <option value="Shahdol">शहडोल (Shahdol)</option>
                        <option value="Khandwa">खंडवा (East Nimar)</option>
                        <option value="Khargone">खरगोन (West Nimar)</option>
                    </select>
                </div>

                <!-- Fertilizer Filter -->
                <div>
                    <label for="fertilizerFilter" class="block text-xs font-semibold text-slate-500 mb-1">खाद का प्रकार (Fertilizer)</label>
                    <select id="fertilizerFilter" onchange="applyFilters()"
                        class="w-full px-3.5 py-2.5 bg-slate-50 border border-slate-200 rounded-xl text-sm font-medium text-slate-700 focus:outline-none focus:ring-2 focus:ring-agri-500 focus:bg-white transition-all cursor-pointer">
                        <option value="ALL">सभी खाद (All Fertilizers)</option>
                        <option value="urea">यूरिया (Urea)</option>
                        <option value="dap">डीएपी (DAP)</option>
                        <option value="npk">एनपीके (NPK)</option>
                        <option value="ssp">सुपर खाद (SSP)</option>
                    </select>
                </div>

                <!-- Availability Status Filter -->
                <div>
                    <label for="statusFilter" class="block text-xs font-semibold text-slate-500 mb-1">स्टॉक की स्थिति</label>
                    <select id="statusFilter" onchange="applyFilters()"
                        class="w-full px-3.5 py-2.5 bg-slate-50 border border-slate-200 rounded-xl text-sm font-medium text-slate-700 focus:outline-none focus:ring-2 focus:ring-agri-500 focus:bg-white transition-all cursor-pointer">
                        <option value="ALL">सभी स्थिति</option>
                        <option value="AVAILABLE">पर्याप्त उपलब्ध (Available)</option>
                        <option value="LOW">कम स्टॉक (Low Stock)</option>
                        <option value="OUT">स्टॉक समाप्त (Out of Stock)</option>
                    </select>
                </div>
            </div>
        </section>

        <!-- Centers Stock Cards Grid -->
        <section aria-label="सहकारी समिति व खाद वितरण केंद्र">
            <div id="centersGrid" class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-5">
                <!-- Cards dynamically rendered via JS -->
            </div>

            <!-- Empty State when no filter matches -->
            <div id="emptyState" class="hidden bg-white rounded-2xl border border-dashed border-slate-300 p-10 text-center space-y-3 mt-4">
                <div class="w-14 h-14 rounded-full bg-slate-100 text-slate-400 flex items-center justify-center mx-auto">
                    <i data-lucide="search-x" class="w-7 h-7"></i>
                </div>
                <h4 class="text-lg font-bold text-slate-800">कोई सहकारी समिति नहीं मिली</h4>
                <p class="text-sm text-slate-500 max-w-md mx-auto">
                    चुने गए फ़िल्टर या खोज शब्द के अनुसार कोई समिति नहीं मिली। कृपया फ़िल्टर बदलें या रीसेट करें।
                </p>
                <button onclick="resetFilters()" class="inline-flex items-center gap-2 px-4 py-2 rounded-xl text-sm font-semibold text-white bg-agri-600 hover:bg-agri-700 transition-colors cursor-pointer">
                    <i data-lucide="rotate-ccw" class="w-4 h-4"></i> सभी फ़िल्टर रीसेट करें
                </button>
            </div>
        </section>
    </main>

    <!-- Footer -->
    <footer class="bg-white border-t border-slate-200 mt-12">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-6 flex flex-col sm:flex-row items-center justify-between gap-4 text-xs sm:text-sm text-slate-500">
            <p>
                <strong class="text-slate-700">X2A Khadya Stock Checker</strong> • संचालक: <strong class="text-agri-700">MR.INDIAN X2A (YT CHANNEL)</strong>
            </p>
            <p class="text-slate-400 text-center sm:text-right">
                नोट: यह पोर्टल मध्य प्रदेश की सहकारी समितियों के खाद स्टॉक की जानकारी हेतु है।
            </p>
        </div>
    </footer>

    <!-- Toast Notification Banner -->
    <div id="toastNotification" class="fixed bottom-20 md:bottom-5 right-5 z-50 transform translate-y-20 opacity-0 transition-all duration-300 pointer-events-none">
        <div class="bg-slate-900 text-white px-4 py-3 rounded-2xl shadow-xl border border-slate-700 flex items-center gap-2.5 text-sm font-medium">
            <i data-lucide="check-circle-2" class="w-4 h-4 text-agri-500 shrink-0"></i>
            <span id="toastMessage">अपडेट सहेजा गया!</span>
        </div>
    </div>

    <!-- ===================================================================== -->
    <!-- MODAL 3: GOVT MRP RATE LIST & LAND FERTILIZER CALCULATOR              -->
    <!-- ===================================================================== -->
    <div id="calculatorModal" class="hidden fixed inset-0 z-50 overflow-y-auto bg-slate-900/60 backdrop-blur-sm flex items-center justify-center p-4">
        <div class="bg-white rounded-3xl max-w-xl w-full shadow-2xl border border-slate-200 overflow-hidden my-8">
            <div class="bg-slate-900 px-6 py-4 text-white flex items-center justify-between">
                <div class="flex items-center gap-2.5">
                    <i data-lucide="calculator" class="w-5 h-5 text-agri-400"></i>
                    <div>
                        <h3 class="font-bold text-base sm:text-lg">सरकारी खाद रेट लिस्ट व खेत कैलकुलेटर</h3>
                        <p class="text-xs text-slate-400">मध्य प्रदेश शासन निर्धारित अधिकतम खुदरा मूल्य (MRP)</p>
                    </div>
                </div>
                <button onclick="closeCalculatorModal()" class="w-8 h-8 rounded-full bg-white/10 hover:bg-white/20 flex items-center justify-center transition-colors cursor-pointer">
                    <i data-lucide="x" class="w-5 h-5"></i>
                </button>
            </div>

            <div class="p-6 space-y-5">
                <div>
                    <h4 class="text-xs font-bold uppercase tracking-wider text-slate-500 mb-2.5">आज की सरकारी खाद रेट लिस्ट (प्रति कट्टा MRP):</h4>
                    <div class="grid grid-cols-2 sm:grid-cols-4 gap-2.5">
                        <div class="p-3 rounded-2xl bg-blue-50 border border-blue-200 text-center">
                            <span class="text-xs font-bold text-blue-900 block">यूरिया (45kg)</span>
                            <span class="text-lg font-extrabold text-slate-900 mt-1 block">₹266.50</span>
                        </div>
                        <div class="p-3 rounded-2xl bg-emerald-50 border border-emerald-200 text-center">
                            <span class="text-xs font-bold text-emerald-900 block">डीएपी (50kg)</span>
                            <span class="text-lg font-extrabold text-slate-900 mt-1 block">₹1,350</span>
                        </div>
                        <div class="p-3 rounded-2xl bg-amber-50 border border-amber-200 text-center">
                            <span class="text-xs font-bold text-amber-900 block">एनपीके (50kg)</span>
                            <span class="text-lg font-extrabold text-slate-900 mt-1 block">₹1,470</span>
                        </div>
                        <div class="p-3 rounded-2xl bg-purple-50 border border-purple-200 text-center">
                            <span class="text-xs font-bold text-purple-900 block">सुपर SSP (50kg)</span>
                            <span class="text-lg font-extrabold text-slate-900 mt-1 block">₹500</span>
                        </div>
                    </div>
                </div>

                <div class="bg-slate-50 rounded-2xl p-4 border border-slate-200 space-y-3.5">
                    <h4 class="text-sm font-extrabold text-slate-800 flex items-center gap-2">
                        <i data-lucide="sprout" class="w-4 h-4 text-agri-600"></i>
                        <span>खेत के रकबे के हिसाब से खाद की मात्रा निकालें</span>
                    </h4>
                    <div class="grid grid-cols-1 sm:grid-cols-3 gap-3">
                        <div>
                            <label class="block text-xs font-bold text-slate-600 mb-1">जमीन का रकबा</label>
                            <input type="number" id="calcLandArea" value="5" min="0.5" step="0.5" oninput="calculateFertilizerNeed()" class="w-full px-3 py-2 bg-white border border-slate-300 rounded-xl text-sm font-bold">
                        </div>
                        <div>
                            <label class="block text-xs font-bold text-slate-600 mb-1">इकाई (Unit)</label>
                            <select id="calcLandUnit" onchange="calculateFertilizerNeed()" class="w-full px-3 py-2 bg-white border border-slate-300 rounded-xl text-sm font-semibold">
                                <option value="bigha">बीघा (Bigha - MP)</option>
                                <option value="acre">एकड़ (Acre)</option>
                                <option value="hectare">हेक्टेयर (Hectare)</option>
                            </select>
                        </div>
                        <div>
                            <label class="block text-xs font-bold text-slate-600 mb-1">फसल (Crop)</label>
                            <select id="calcCropType" onchange="calculateFertilizerNeed()" class="w-full px-3 py-2 bg-white border border-slate-300 rounded-xl text-sm font-semibold">
                                <option value="wheat">गेहूँ (Wheat)</option>
                                <option value="gram">चना / मसूर (Gram)</option>
                                <option value="soybean">सोयाबीन / मक्का</option>
                                <option value="garlic">लहसुन / प्याज</option>
                            </select>
                        </div>
                    </div>

                    <div id="calculatorResultBox" class="bg-white rounded-xl p-3.5 border border-agri-200 space-y-2"></div>
                </div>
            </div>
        </div>
    </div>

    <!-- ===================================================================== -->
    <!-- MODAL 1: CENTER DETAILS & SHARE MODAL                                 -->
    <!-- ===================================================================== -->
    <div id="detailsModal" class="hidden fixed inset-0 z-50 overflow-y-auto bg-slate-900/60 backdrop-blur-sm flex items-center justify-center p-4">
        <div class="bg-white rounded-3xl max-w-lg w-full shadow-2xl border border-slate-200 overflow-hidden my-8">
            <div class="bg-gradient-to-r from-agri-700 to-agri-800 px-6 py-4 text-white flex items-center justify-between">
                <div class="flex items-center gap-2.5">
                    <i data-lucide="building-2" class="w-5 h-5 text-agri-200"></i>
                    <h3 id="modalDetailTitle" class="font-bold text-base sm:text-lg">केंद्र का विवरण</h3>
                </div>
                <button onclick="closeDetailsModal()" class="w-8 h-8 rounded-full bg-white/10 hover:bg-white/20 flex items-center justify-center transition-colors cursor-pointer">
                    <i data-lucide="x" class="w-5 h-5"></i>
                </button>
            </div>
            <div id="modalDetailBody" class="p-6 text-sm text-slate-700 space-y-4"></div>
            <div class="px-6 py-4 bg-slate-50 border-t border-slate-200 flex flex-wrap items-center justify-between gap-2">
                <button onclick="copyCenterShareText()" class="inline-flex items-center gap-1.5 px-4 py-2 rounded-xl text-xs sm:text-sm font-bold text-agri-800 bg-agri-100 hover:bg-agri-200 transition-colors cursor-pointer">
                    <i data-lucide="share-2" class="w-4 h-4"></i>
                    <span>WhatsApp / जानकारी कॉपी करें</span>
                </button>
                <button onclick="closeDetailsModal()" class="px-4 py-2 rounded-xl text-xs sm:text-sm font-bold text-white bg-slate-800 hover:bg-slate-700 transition-colors cursor-pointer">
                    बंद करें
                </button>
            </div>
        </div>
    </div>

    <!-- ===================================================================== -->
    <!-- MODAL 2: ADMIN STOCK UPDATE & ADD SOCIETY MODAL                       -->
    <!-- ===================================================================== -->
    <div id="adminModal" class="hidden fixed inset-0 z-50 overflow-y-auto bg-slate-900/60 backdrop-blur-sm flex items-center justify-center p-4">
        <div class="bg-white rounded-3xl max-w-xl w-full shadow-2xl border border-slate-200 overflow-hidden my-8">
            <div class="bg-slate-900 px-6 py-4 text-white flex items-center justify-between">
                <div class="flex items-center gap-2.5">
                    <i data-lucide="shield-check" class="w-5 h-5 text-saffron-500"></i>
                    <div>
                        <h3 class="font-bold text-base sm:text-lg">Admin स्टॉक अपडेट पैनल (MP)</h3>
                        <p class="text-xs text-slate-400">संचालक: MR.INDIAN X2A (YT CHANNEL)</p>
                    </div>
                </div>
                <button onclick="closeAdminModal()" class="w-8 h-8 rounded-full bg-white/10 hover:bg-white/20 flex items-center justify-center transition-colors cursor-pointer">
                    <i data-lucide="x" class="w-5 h-5"></i>
                </button>
            </div>

            <!-- Admin Password Gate Form -->
            <form id="adminAuthForm" onsubmit="handleAdminPasswordSubmit(event)" class="p-6 space-y-4">
                <div class="bg-amber-50 border border-amber-200 rounded-2xl p-3.5 text-xs text-amber-900 flex items-start gap-2.5">
                    <i data-lucide="lock" class="w-4 h-4 text-amber-600 shrink-0 mt-0.5"></i>
                    <span>स्टॉक अपडेट करने या नई समिति जोड़ने के लिए कृपया Admin पासवर्ड दर्ज करें।</span>
                </div>

                <div>
                    <label for="adminPasswordInput" class="block text-xs font-bold text-slate-700 mb-1">Admin पासवर्ड *</label>
                    <div class="relative flex items-center">
                        <input type="password" id="adminPasswordInput" required placeholder="पासवर्ड दर्ज करें..."
                            class="w-full pl-3.5 pr-11 py-2.5 bg-slate-50 border border-slate-300 rounded-xl text-sm font-semibold focus:outline-none focus:ring-2 focus:ring-agri-500 focus:bg-white">
                        <button type="button" onclick="toggleAdminPasswordVisibility()" class="absolute right-2.5 text-slate-500 hover:text-slate-800 cursor-pointer" title="पासवर्ड देखें/छुपाएं">
                            <i id="adminPassEyeIcon" data-lucide="eye" class="w-4 h-4"></i>
                        </button>
                    </div>
                    <p id="adminAuthError" class="hidden text-xs font-bold text-rose-600 mt-1.5">
                        गलत पासवर्ड! कृपया सही पासवर्ड डालें।
                    </p>
                </div>

                <div class="flex items-center justify-end gap-2 pt-2 border-t border-slate-100">
                    <button type="button" onclick="closeAdminModal()" class="px-4 py-2.5 rounded-xl text-xs sm:text-sm font-semibold text-slate-600 bg-slate-100 hover:bg-slate-200 cursor-pointer">
                        रद्द करें
                    </button>
                    <button type="submit" class="inline-flex items-center gap-1.5 px-5 py-2.5 rounded-xl text-xs sm:text-sm font-bold text-white bg-slate-900 hover:bg-slate-800 cursor-pointer">
                        <i data-lucide="unlock" class="w-4 h-4 text-saffron-500"></i>
                        <span>अनलॉक करें</span>
                    </button>
                </div>
            </form>

            <!-- Admin Tabs -->
            <div id="adminTabsBar" class="hidden border-b border-slate-200 px-6 pt-3 flex gap-4 text-sm font-bold">
                <button type="button" id="tabUpdateStock" onclick="switchAdminTab('update')" class="pb-2.5 border-b-2 border-agri-600 text-agri-700 cursor-pointer">
                    मौजूदा केंद्र का स्टॉक बदलें
                </button>
                <button type="button" id="tabAddCenter" onclick="switchAdminTab('add')" class="pb-2.5 border-b-2 border-transparent text-slate-500 hover:text-slate-800 cursor-pointer">
                    + नई समिति जोड़ें
                </button>
            </div>

            <!-- Tab 1: Update Existing Center Stock -->
            <form id="adminUpdateForm" onsubmit="handleUpdateStockSubmit(event)" class="hidden p-6 space-y-4">
                <div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
                    <div>
                        <label for="adminDistrictFilter" class="block text-xs font-bold text-slate-700 mb-1">जिला फ़िल्टर करें</label>
                        <select id="adminDistrictFilter" onchange="filterAdminCentersByDistrict()" class="w-full px-3 py-2.5 bg-slate-50 border border-slate-300 rounded-xl text-sm font-semibold focus:outline-none focus:ring-2 focus:ring-agri-500">
                        </select>
                    </div>
                    <div>
                        <label for="adminCenterSelect" class="block text-xs font-bold text-slate-700 mb-1">सहकारी समिति चुनें</label>
                        <select id="adminCenterSelect" onchange="populateAdminForm()" class="w-full px-3 py-2.5 bg-slate-50 border border-slate-300 rounded-xl text-sm font-semibold focus:outline-none focus:ring-2 focus:ring-agri-500">
                        </select>
                    </div>
                </div>

                <div class="grid grid-cols-2 gap-3.5">
                    <div>
                        <label for="editUrea" class="block text-xs font-bold text-blue-900 mb-1">यूरिया (Urea - कट्टे)</label>
                        <input type="number" id="editUrea" min="0" required class="w-full px-3.5 py-2 bg-blue-50/40 border border-blue-200 rounded-xl text-sm font-bold text-slate-900 focus:outline-none focus:ring-2 focus:ring-blue-500">
                    </div>
                    <div>
                        <label for="editDap" class="block text-xs font-bold text-emerald-900 mb-1">डीएपी (DAP - कट्टे)</label>
                        <input type="number" id="editDap" min="0" required class="w-full px-3.5 py-2 bg-emerald-50/40 border border-emerald-200 rounded-xl text-sm font-bold text-slate-900 focus:outline-none focus:ring-2 focus:ring-emerald-500">
                    </div>
                    <div>
                        <label for="editNpk" class="block text-xs font-bold text-amber-900 mb-1">एनपीके (NPK - कट्टे)</label>
                        <input type="number" id="editNpk" min="0" required class="w-full px-3.5 py-2 bg-amber-50/40 border border-amber-200 rounded-xl text-sm font-bold text-slate-900 focus:outline-none focus:ring-2 focus:ring-amber-500">
                    </div>
                    <div>
                        <label for="editSsp" class="block text-xs font-bold text-purple-900 mb-1">सुपर खाद (SSP - कट्टे)</label>
                        <input type="number" id="editSsp" min="0" required class="w-full px-3.5 py-2 bg-purple-50/40 border border-purple-200 rounded-xl text-sm font-bold text-slate-900 focus:outline-none focus:ring-2 focus:ring-purple-500">
                    </div>
                </div>

                <div>
                    <label for="editRemarks" class="block text-xs font-bold text-slate-700 mb-1">सूचना / टिप्पणी (Optional)</label>
                    <input type="text" id="editRemarks" placeholder="जैसे: नया रैक आया है या टोकन जरूरी है..." class="w-full px-3.5 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-agri-500">
                </div>

                <div class="flex flex-wrap items-center justify-between gap-2 pt-3 border-t border-slate-100">
                    <button type="button" onclick="resetDemoData()" class="inline-flex items-center gap-1 text-xs font-bold text-rose-600 hover:text-rose-700 px-2.5 py-2 rounded-lg hover:bg-rose-50 cursor-pointer">
                        <i data-lucide="rotate-ccw" class="w-3.5 h-3.5"></i> डिफ़ॉल्ट डेटा रीसेट
                    </button>
                    <div class="flex items-center gap-2">
                        <button type="button" onclick="closeAdminModal()" class="px-4 py-2 rounded-xl text-xs sm:text-sm font-semibold text-slate-600 bg-slate-100 hover:bg-slate-200 cursor-pointer">
                            रद्द करें
                        </button>
                        <button type="submit" class="inline-flex items-center gap-1.5 px-5 py-2 rounded-xl text-xs sm:text-sm font-bold text-white bg-agri-600 hover:bg-agri-700 shadow-sm cursor-pointer">
                            <i data-lucide="save" class="w-4 h-4"></i> स्टॉक सेव करें
                        </button>
                    </div>
                </div>
            </form>

            <!-- Tab 2: Add New Society Center -->
            <form id="adminAddForm" onsubmit="handleAddCenterSubmit(event)" class="hidden p-6 space-y-3.5">
                <div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
                    <div>
                        <label for="newCenterName" class="block text-xs font-bold text-slate-700 mb-1">समिति / केंद्र का नाम *</label>
                        <input type="text" id="newCenterName" required placeholder="जैसे: सारंगपुर सहकारी समिति" class="w-full px-3 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-agri-500">
                    </div>
                    <div>
                        <label for="newCenterDistrict" class="block text-xs font-bold text-slate-700 mb-1">जिला (MP District) *</label>
                        <select id="newCenterDistrict" required class="w-full px-3 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm font-semibold focus:outline-none focus:ring-2 focus:ring-agri-500">
                            <option value="Rajgarh">राजगढ़ (Rajgarh)</option>
                            <option value="Indore">इंदौर (Indore)</option>
                            <option value="Ujjain">उज्जैन (Ujjain)</option>
                            <option value="Bhopal">भोपाल (Bhopal)</option>
                            <option value="Sehore">सीहोर (Sehore)</option>
                            <option value="Dewas">देवास (Dewas)</option>
                            <option value="Shajapur">शाजापुर (Shajapur)</option>
                            <option value="Agar Malwa">आगर मालवा (Agar Malwa)</option>
                            <option value="Vidisha">विदिशा (Vidisha)</option>
                            <option value="Raisen">रायसेन (Raisen)</option>
                            <option value="Mandsaur">मंदसौर (Mandsaur)</option>
                            <option value="Neemuch">नीमच (Neemuch)</option>
                            <option value="Ratlam">रतलाम (Ratlam)</option>
                            <option value="Gwalior">ग्वालियर (Gwalior)</option>
                            <option value="Morena">मुरैना (Morena)</option>
                            <option value="Shivpuri">शिवपुरी (Shivpuri)</option>
                            <option value="Guna">गुना (Guna)</option>
                            <option value="Sagar">सागर (Sagar)</option>
                            <option value="Damoh">दमोह (Damoh)</option>
                            <option value="Chhatarpur">छतरपुर (Chhatarpur)</option>
                            <option value="Tikamgarh">टीकमगढ़ / निवाड़ी</option>
                            <option value="Jabalpur">जबलपुर (Jabalpur)</option>
                            <option value="Chhindwara">छिंदवाड़ा / पांढुर्णा</option>
                            <option value="Seoni">सिवनी (Seoni)</option>
                            <option value="Narmadapuram">नर्मदापुरम</option>
                            <option value="Rewa">रीवा / मऊगंज</option>
                            <option value="Satna">सतना / मैहर</option>
                            <option value="Shahdol">शहडोल</option>
                            <option value="Khandwa">खंडवा</option>
                            <option value="Khargone">खरगोन</option>
                        </select>
                    </div>
                </div>

                <div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
                    <div>
                        <label for="newCenterTehsil" class="block text-xs font-bold text-slate-700 mb-1">तहसील / क्षेत्र *</label>
                        <input type="text" id="newCenterTehsil" required placeholder="जैसे: सारंगपुर" class="w-full px-3 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-agri-500">
                    </div>
                    <div>
                        <label for="newCenterPhone" class="block text-xs font-bold text-slate-700 mb-1">संपर्क मोबाइल नंबर</label>
                        <input type="text" id="newCenterPhone" placeholder="जैसे: 9826000000" class="w-full px-3 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-agri-500">
                    </div>
                </div>

                <div>
                    <label for="newCenterAddress" class="block text-xs font-bold text-slate-700 mb-1">पूरा पता *</label>
                    <input type="text" id="newCenterAddress" required placeholder="ग्राम / वार्ड, तहसील, जिला, मध्य प्रदेश" class="w-full px-3 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-agri-500">
                </div>

                <div class="grid grid-cols-2 sm:grid-cols-4 gap-2.5">
                    <div>
                        <label for="newUrea" class="block text-xs font-bold text-blue-900 mb-1">यूरिया</label>
                        <input type="number" id="newUrea" min="0" value="100" required class="w-full px-2.5 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm font-bold">
                    </div>
                    <div>
                        <label for="newDap" class="block text-xs font-bold text-emerald-900 mb-1">डीएपी</label>
                        <input type="number" id="newDap" min="0" value="50" required class="w-full px-2.5 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm font-bold">
                    </div>
                    <div>
                        <label for="newNpk" class="block text-xs font-bold text-amber-900 mb-1">NPK</label>
                        <input type="number" id="newNpk" min="0" value="40" required class="w-full px-2.5 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm font-bold">
                    </div>
                    <div>
                        <label for="newSsp" class="block text-xs font-bold text-purple-900 mb-1">SSP</label>
                        <input type="number" id="newSsp" min="0" value="60" required class="w-full px-2.5 py-2 bg-slate-50 border border-slate-300 rounded-xl text-sm font-bold">
                    </div>
                </div>

                <div class="flex items-center justify-end gap-2 pt-3 border-t border-slate-100">
                    <button type="button" onclick="closeAdminModal()" class="px-4 py-2 rounded-xl text-xs sm:text-sm font-semibold text-slate-600 bg-slate-100 hover:bg-slate-200 cursor-pointer">
                        रद्द करें
                    </button>
                    <button type="submit" class="inline-flex items-center gap-1.5 px-5 py-2 rounded-xl text-xs sm:text-sm font-bold text-white bg-agri-600 hover:bg-agri-700 shadow-sm cursor-pointer">
                        <i data-lucide="plus-circle" class="w-4 h-4"></i> समिति जोड़ें
                    </button>
                </div>
            </form>
        </div>
    </div>

    <!-- ===================================================================== -->
    <!-- MODAL 5: MY SAVED E-TOKENS HISTORY MODAL                              -->
    <!-- ===================================================================== -->
    <div id="myTokensModal" class="hidden fixed inset-0 z-50 overflow-y-auto bg-slate-900/60 backdrop-blur-sm flex items-center justify-center p-4">
        <div class="bg-white rounded-3xl max-w-lg w-full shadow-2xl border border-slate-200 overflow-hidden my-8">
            <div class="bg-amber-600 px-6 py-4 text-white flex items-center justify-between">
                <div class="flex items-center gap-2">
                    <i data-lucide="ticket" class="w-5 h-5"></i>
                    <h3 class="font-bold text-base sm:text-lg">मेरे बुक किए गए ई-टोकन (My Tokens)</h3>
                </div>
                <button onclick="closeMyTokensModal()" class="w-8 h-8 rounded-full bg-white/15 hover:bg-white/25 flex items-center justify-center transition-colors cursor-pointer">
                    <i data-lucide="x" class="w-5 h-5"></i>
                </button>
            </div>
            <div id="myTokensListContainer" class="p-6 max-h-[65vh] overflow-y-auto space-y-3"></div>
        </div>
    </div>

    <!-- Mobile Sticky Bottom Quick Navigation Bar -->
    <nav class="md:hidden fixed bottom-0 inset-x-0 bg-white border-t border-slate-200 z-40 px-2 py-1.5 flex items-center justify-around shadow-lg">
        <button onclick="resetFilters(); window.scrollTo({top: 0, behavior: 'smooth'})" class="flex flex-col items-center gap-0.5 px-3 py-1 text-agri-700 font-bold text-[11px] cursor-pointer">
            <i data-lucide="home" class="w-5 h-5"></i>
            <span>होम</span>
        </button>
        <button onclick="openCalculatorModal()" class="flex flex-col items-center gap-0.5 px-3 py-1 text-slate-600 hover:text-agri-700 font-semibold text-[11px] cursor-pointer">
            <i data-lucide="calculator" class="w-5 h-5"></i>
            <span>रेट/कैलकुलेटर</span>
        </button>
        <button onclick="openAdminModal()" class="flex flex-col items-center gap-0.5 px-3 py-1 text-slate-600 hover:text-slate-900 font-semibold text-[11px] cursor-pointer">
            <i data-lucide="shield-check" class="w-5 h-5"></i>
            <span>Admin</span>
        </button>
    </nav>

    <!-- ===================================================================== -->
    <!-- APPLICATION LOGIC (JavaScript) -->
    <!-- ===================================================================== -->
    <script>
        const STORAGE_KEY = 'x2a_khadya_stock_mp_v3';
        const FAV_STORAGE_KEY = 'x2a_fav_societies_v1';
        const ADMIN_PASSCODE = 'Hellomy125';

        let isAdminUnlocked = false;
        const FERT_MRP = {
            urea: { name: 'यूरिया (Urea 45kg)', price: 266.5 },
            dap: { name: 'डीएपी (DAP 50kg)', price: 1350 },
            npk: { name: 'एनपीके (NPK 12:32:16)', price: 1470 },
            ssp: { name: 'सुपर खाद (SSP 50kg)', price: 500 }
        };

        const DISTRICT_HINDI_MAP = {
            'Rajgarh': 'राजगढ़',
            'Indore': 'इंदौर',
            'Ujjain': 'उज्जैन',
            'Bhopal': 'भोपाल',
            'Sehore': 'सीहोर',
            'Dewas': 'देवास',
            'Shajapur': 'शाजापुर',
            'Agar Malwa': 'आगर मालवा',
            'Vidisha': 'विदिशा',
            'Raisen': 'रायसेन',
            'Mandsaur': 'मंदसौर',
            'Neemuch': 'नीमच',
            'Ratlam': 'रतलाम',
            'Gwalior': 'ग्वालियर',
            'Morena': 'मुरैना',
            'Shivpuri': 'शिवपुरी',
            'Guna': 'गुना',
            'Sagar': 'सागर',
            'Damoh': 'दमोह',
            'Chhatarpur': 'छतरपुर',
            'Tikamgarh': 'टीकमगढ़ / निवाड़ी',
            'Jabalpur': 'जबलपुर',
            'Chhindwara': 'छिंदवाड़ा / पांढुर्णा',
            'Seoni': 'सिवनी',
            'Narmadapuram': 'नर्मदापुरम',
            'Rewa': 'रीवा / मऊगंज',
            'Satna': 'सतना / मैहर',
            'Shahdol': 'शहडोल',
            'Khandwa': 'खंडवा',
            'Khargone': 'खरगोन'
        };

        const DEFAULT_MP_CENTERS = [
            {
                id: 'mp_raj_chatukheda',
                name: 'चटुखेड़ा सहकारी समिति (Chatukheda Society)',
                district: 'Rajgarh',
                districtHi: 'राजगढ़',
                tehsil: 'चटुखेड़ा क्षेत्र',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'ग्राम चटुखेड़ा, जिला राजगढ़, मध्य प्रदेश',
                phone: '9826101001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 420, dap: 195, npk: 130, ssp: 240 },
                remarks: 'चटुखेड़ा समिति में यूरिया व डीएपी उपलब्ध; आधार कार्ड व पावती (B1) साथ लाएं।',
                lastUpdated: 'आज, 09:45 AM'
            },
            {
                id: 'mp_raj_01',
                name: 'राजगढ़ केंद्रीय सहकारी विपणन केंद्र',
                district: 'Rajgarh',
                districtHi: 'राजगढ़',
                tehsil: 'राजगढ़',
                type: 'सहकारी विपणन संघ',
                address: 'मंडी रोड, राजगढ़, मध्य प्रदेश',
                phone: '9826101002',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 310, dap: 140, npk: 85, ssp: 160 },
                remarks: 'ई-टोकन के माध्यम से नियमित खाद वितरण जारी।',
                lastUpdated: 'आज, 09:15 AM'
            },
            {
                id: 'mp_raj_02',
                name: 'ब्यावरा सहकारी समिति (Biaora PACS)',
                district: 'Rajgarh',
                districtHi: 'राजगढ़',
                tehsil: 'ब्यावरा',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'एबी रोड, ब्यावरा, जिला राजगढ़, म.प्र.',
                phone: '9826101003',
                timings: 'सोम - शनि, सुबह 10:30 से शाम 05:00',
                stock: { urea: 260, dap: 95, npk: 60, ssp: 120 },
                remarks: 'किसानों के लिए पीओएस (POS) मशीन से वितरण उपलब्ध।',
                lastUpdated: 'आज, 08:50 AM'
            },
            {
                id: 'mp_raj_03',
                name: 'नरसिंहगढ़ सहकारी समिति',
                district: 'Rajgarh',
                districtHi: 'राजगढ़',
                tehsil: 'नरसिंहगढ़',
                type: 'सहकारी समिति केंद्र',
                address: 'बस स्टैंड के पास, नरसिंहगढ़, जिला राजगढ़, म.प्र.',
                phone: '9826101004',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 04:30',
                stock: { urea: 185, dap: 42, npk: 55, ssp: 90 },
                remarks: 'डीएपी का स्टॉक सीमित है, शीघ्र संपर्क करें।',
                lastUpdated: 'कल, 05:30 PM'
            },
            {
                id: 'mp_raj_04',
                name: 'खिलचीपुर व जीरापुर सहकारी समिति',
                district: 'Rajgarh',
                districtHi: 'राजगढ़',
                tehsil: 'खिलचीपुर / जीरापुर',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'मुख्य बाजार मार्ग, खिलचीपुर, जिला राजगढ़, म.प्र.',
                phone: '9826101005',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 220, dap: 110, npk: 70, ssp: 145 },
                remarks: 'सभी पंजीकृत किसानों को नियमानुसार खाद दिया जा रहा है।',
                lastUpdated: 'आज, 08:20 AM'
            },
            {
                id: 'mp_ind_01',
                name: 'इंदौर सहकारी विपणन समिति (सांवेर रोड)',
                district: 'Indore',
                districtHi: 'इंदौर',
                tehsil: 'इंदौर / सांवेर',
                type: 'MP Markfed / सहकारी केंद्र',
                address: 'सांवेर रोड, इंदौर, मध्य प्रदेश',
                phone: '9827020001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:30',
                stock: { urea: 540, dap: 230, npk: 180, ssp: 310 },
                remarks: 'यूरिया और डीएपी का पर्याप्त स्टॉक उपलब्ध है।',
                lastUpdated: 'आज, 09:40 AM'
            },
            {
                id: 'mp_ind_02',
                name: 'महू (डॉ. आंबेडकर नगर) सहकारी समिति',
                district: 'Indore',
                districtHi: 'इंदौर',
                tehsil: 'महू / देपालपुर',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'मंडी क्षेत्र, महू, जिला इंदौर, म.प्र.',
                phone: '9827020002',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 190, dap: 78, npk: 95, ssp: 130 },
                remarks: 'भू-अभिलेख पावती व आधार अनिवार्य।',
                lastUpdated: 'आज, 08:30 AM'
            },
            {
                id: 'mp_ujj_01',
                name: 'उज्जैन केंद्रीय सहकारी विपणन केंद्र',
                district: 'Ujjain',
                districtHi: 'उज्जैन',
                tehsil: 'उज्जैन',
                type: 'सहकारी विपणन संघ',
                address: 'मक्सी रोड, उज्जैन, मध्य प्रदेश',
                phone: '9827030001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 460, dap: 210, npk: 140, ssp: 220 },
                remarks: 'उज्जैन व घट्टिया क्षेत्र के किसानों हेतु वितरण केंद्र।',
                lastUpdated: 'आज, 09:10 AM'
            },
            {
                id: 'mp_ujj_02',
                name: 'नागदा व महिदपुर सहकारी समिति',
                district: 'Ujjain',
                districtHi: 'उज्जैन',
                tehsil: 'नागदा / महिदपुर',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'कृषि उपज मंडी के पास, नागदा, जिला उज्जैन, म.प्र.',
                phone: '9827030002',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 04:30',
                stock: { urea: 155, dap: 35, npk: 50, ssp: 85 },
                remarks: 'डीएपी का कम स्टॉक शेष है।',
                lastUpdated: 'कल, 06:00 PM'
            },
            {
                id: 'mp_bho_01',
                name: 'भोपाल सहकारी विपणन संघ केंद्र (करोंद)',
                district: 'Bhopal',
                districtHi: 'भोपाल',
                tehsil: 'हुजूर / बैरसिया',
                type: 'MP Markfed / सहकारी केंद्र',
                address: 'करोंद चौराहा, भोपाल, मध्य प्रदेश',
                phone: '9826040001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:30',
                stock: { urea: 390, dap: 175, npk: 125, ssp: 195 },
                remarks: 'ऑनलाइन/ऑफलाइन टोकन से खाद वितरण जारी।',
                lastUpdated: 'आज, 09:25 AM'
            },
            {
                id: 'mp_seh_01',
                name: 'सीहोर व आष्टा सहकारी समिति',
                district: 'Sehore',
                districtHi: 'सीहोर',
                tehsil: 'सीहोर / आष्टा',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'इंदौर-भोपाल रोड, सीहोर, मध्य प्रदेश',
                phone: '9826050001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 340, dap: 150, npk: 110, ssp: 175 },
                remarks: 'सीहोर व इछावर के किसानों के लिए नियमित स्टॉक उपलब्ध।',
                lastUpdated: 'आज, 08:45 AM'
            },
            {
                id: 'mp_dew_01',
                name: 'देवास सहकारी विपणन समिति',
                district: 'Dewas',
                districtHi: 'देवास',
                tehsil: 'देवास / सोनकच्छ',
                type: 'सहकारी विपणन संघ',
                address: 'एबी रोड, देवास, मध्य प्रदेश',
                phone: '9827060001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 295, dap: 125, npk: 90, ssp: 150 },
                remarks: 'यूरिया, डीएपी व एनपीके उपलब्ध।',
                lastUpdated: 'आज, 09:00 AM'
            },
            {
                id: 'mp_shj_01',
                name: 'शाजापुर व शुजालपुर सहकारी समिति',
                district: 'Shajapur',
                districtHi: 'शाजापुर',
                tehsil: 'शाजापुर / शुजालपुर',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'मंडी रोड, शाजापुर, मध्य प्रदेश',
                phone: '9826070001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 275, dap: 115, npk: 80, ssp: 140 },
                remarks: 'रबी सीजन हेतु खाद वितरण सुचारू।',
                lastUpdated: 'आज, 08:15 AM'
            },
            {
                id: 'mp_agr_01',
                name: 'आगर मालवा व सुसनेर सहकारी समिति',
                district: 'Agar Malwa',
                districtHi: 'आगर मालवा',
                tehsil: 'आगर / सुसनेर',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'छावनी रोड, आगर मालवा, मध्य प्रदेश',
                phone: '9826080001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 04:30',
                stock: { urea: 210, dap: 88, npk: 65, ssp: 115 },
                remarks: 'सदस्य किसानों को प्राथमिकता से वितरण।',
                lastUpdated: 'आज, 08:05 AM'
            },
            {
                id: 'mp_vid_01',
                name: 'विदिशा व गंजबासौदा सहकारी समिति',
                district: 'Vidisha',
                districtHi: 'विदिशा',
                tehsil: 'विदिशा / गंजबासौदा',
                type: 'सहकारी विपणन संघ',
                address: 'सांची रोड, विदिशा, मध्य प्रदेश',
                phone: '9826090001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 360, dap: 185, npk: 105, ssp: 165 },
                remarks: 'डीएपी व यूरिया का स्टॉक उपलब्ध है।',
                lastUpdated: 'आज, 09:20 AM'
            },
            {
                id: 'mp_rai_01',
                name: 'रायसेन व मंडीदीप सहकारी समिति',
                district: 'Raisen',
                districtHi: 'रायसेन',
                tehsil: 'रायसेन / गैरतगंज',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'सागर रोड, रायसेन, मध्य प्रदेश',
                phone: '9826110001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 245, dap: 120, npk: 75, ssp: 110 },
                remarks: 'पावती व आधार कार्ड साथ रखें।',
                lastUpdated: 'कल, 04:45 PM'
            },
            {
                id: 'mp_man_01',
                name: 'मंदसौर सहकारी विपणन केंद्र',
                district: 'Mandsaur',
                districtHi: 'मंदसौर',
                tehsil: 'मंदसौर / दलौदा',
                type: 'सहकारी विपणन संघ',
                address: 'महू-नीमच रोड, मंदसौर, मध्य प्रदेश',
                phone: '9827120001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 330, dap: 145, npk: 120, ssp: 200 },
                remarks: 'एनपीके व यूरिया उपलब्ध।',
                lastUpdated: 'आज, 08:55 AM'
            },
            {
                id: 'mp_nee_01',
                name: 'नीमच सहकारी समिति केंद्र',
                district: 'Neemuch',
                districtHi: 'नीमच',
                tehsil: 'नीमच / जावद',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'मंडी रोड, नीमच, मध्य प्रदेश',
                phone: '9827130001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 205, dap: 90, npk: 70, ssp: 135 },
                remarks: 'नियमित वितरण जारी।',
                lastUpdated: 'आज, 08:40 AM'
            },
            {
                id: 'mp_rat_01',
                name: 'रतलाम व जावरा सहकारी समिति',
                district: 'Ratlam',
                districtHi: 'रतलाम',
                tehsil: 'रतलाम / जावरा',
                type: 'सहकारी विपणन संघ',
                address: 'सैलाना रोड, रतलाम, मध्य प्रदेश',
                phone: '9827140001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 280, dap: 130, npk: 95, ssp: 160 },
                remarks: 'खाद उपलब्धता की जानकारी अपडेट कर दी गई है।',
                lastUpdated: 'आज, 09:05 AM'
            },
            {
                id: 'mp_gwa_01',
                name: 'ग्वालियर सहकारी विपणन समिति',
                district: 'Gwalior',
                districtHi: 'ग्वालियर',
                tehsil: 'ग्वालियर / डबरा',
                type: 'MP Markfed / सहकारी केंद्र',
                address: 'लक्ष्मीगंज मंडी, ग्वालियर, मध्य प्रदेश',
                phone: '9826150001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 410, dap: 170, npk: 115, ssp: 180 },
                remarks: 'यूरिया और डीएपी उपलब्ध।',
                lastUpdated: 'आज, 09:18 AM'
            },
            {
                id: 'mp_mor_01',
                name: 'मुरैना व अंबाह सहकारी समिति',
                district: 'Morena',
                districtHi: 'मुरैना',
                tehsil: 'मुरैना / अंबाह',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'स्टेशन रोड, मुरैना, मध्य प्रदेश',
                phone: '9826160001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 04:30',
                stock: { urea: 290, dap: 110, npk: 65, ssp: 120 },
                remarks: 'सरसों व गेहूँ हेतु खाद वितरण जारी।',
                lastUpdated: 'कल, 05:10 PM'
            },
            {
                id: 'mp_shv_01',
                name: 'शिवपुरी सहकारी समिति केंद्र',
                district: 'Shivpuri',
                districtHi: 'शिवपुरी',
                tehsil: 'शिवपुरी / कोलारस',
                type: 'सहकारी विपणन संघ',
                address: 'झांसी रोड, शिवपुरी, मध्य प्रदेश',
                phone: '9826170001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 235, dap: 95, npk: 60, ssp: 105 },
                remarks: 'आधार व पावती से वितरण।',
                lastUpdated: 'आज, 08:25 AM'
            },
            {
                id: 'mp_gun_01',
                name: 'गुना व राघौगढ़ सहकारी समिति',
                district: 'Guna',
                districtHi: 'गुना',
                tehsil: 'गुना / राघौगढ़',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'एबी रोड, गुना, मध्य प्रदेश',
                phone: '9826180001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 270, dap: 125, npk: 85, ssp: 140 },
                remarks: 'स्टॉक उपलब्ध है।',
                lastUpdated: 'आज, 08:50 AM'
            },
            {
                id: 'mp_sag_01',
                name: 'सागर व बीना सहकारी विपणन केंद्र',
                district: 'Sagar',
                districtHi: 'सागर',
                tehsil: 'सागर / बीना',
                type: 'सहकारी विपणन संघ',
                address: 'मकरोनिया चौराहा, सागर, मध्य प्रदेश',
                phone: '9826190001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 350, dap: 160, npk: 95, ssp: 155 },
                remarks: 'डीएपी व यूरिया का वितरण जारी।',
                lastUpdated: 'आज, 09:12 AM'
            },
            {
                id: 'mp_dam_01',
                name: 'दमोह सहकारी समिति केंद्र',
                district: 'Damoh',
                districtHi: 'दमोह',
                tehsil: 'दमोह / हटा',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'जबलपुर नाका, दमोह, मध्य प्रदेश',
                phone: '9826200001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 04:30',
                stock: { urea: 165, dap: 45, npk: 40, ssp: 80 },
                remarks: 'डीएपी का स्टॉक कम है।',
                lastUpdated: 'कल, 04:20 PM'
            },
            {
                id: 'mp_chh_01',
                name: 'छतरपुर व नौगांव सहकारी समिति',
                district: 'Chhatarpur',
                districtHi: 'छतरपुर',
                tehsil: 'छतरपुर / नौगांव',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'पन्ना रोड, छतरपुर, मध्य प्रदेश',
                phone: '9826210001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 195, dap: 85, npk: 55, ssp: 100 },
                remarks: 'नियमित वितरण जारी।',
                lastUpdated: 'आज, 08:35 AM'
            },
            {
                id: 'mp_jab_01',
                name: 'जबलपुर सहकारी विपणन समिति',
                district: 'Jabalpur',
                districtHi: 'जबलपुर',
                tehsil: 'जबलपुर / सिहोरा',
                type: 'MP Markfed / सहकारी केंद्र',
                address: 'दमोह रोड, जबलपुर, मध्य प्रदेश',
                phone: '9826220001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:30',
                stock: { urea: 380, dap: 165, npk: 120, ssp: 190 },
                remarks: 'सभी प्रमुख खाद उपलब्ध।',
                lastUpdated: 'आज, 09:30 AM'
            },
            {
                id: 'mp_chd_01',
                name: 'छिंदवाड़ा सहकारी विपणन केंद्र',
                district: 'Chhindwara',
                districtHi: 'छिंदवाड़ा / पांढुर्णा',
                tehsil: 'छिंदवाड़ा / सौंसर',
                type: 'सहकारी विपणन संघ',
                address: 'नरसिंहपुर रोड, छिंदवाड़ा, मध्य प्रदेश',
                phone: '9826230001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 315, dap: 140, npk: 110, ssp: 170 },
                remarks: 'मक्का व रबी फसलों हेतु खाद उपलब्ध।',
                lastUpdated: 'आज, 08:48 AM'
            },
            {
                id: 'mp_nar_01',
                name: 'नर्मदापुरम (होशंगाबाद) व इटारसी समिति',
                district: 'Narmadapuram',
                districtHi: 'नर्मदापुरम',
                tehsil: 'नर्मदापुरम / इटारसी',
                type: 'सहकारी विपणन संघ',
                address: 'मंडी रोड, नर्मदापुरम, मध्य प्रदेश',
                phone: '9826240001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 430, dap: 190, npk: 135, ssp: 210 },
                remarks: 'गेहूँ क्षेत्र के लिए पर्याप्त यूरिया व डीएपी उपलब्ध।',
                lastUpdated: 'आज, 09:22 AM'
            },
            {
                id: 'mp_rew_01',
                name: 'रीवा सहकारी विपणन समिति',
                district: 'Rewa',
                districtHi: 'रीवा / मऊगंज',
                type: 'सहकारी विपणन संघ',
                address: 'सिरमौर चौराहा के पास, रीवा, मध्य प्रदेश',
                phone: '9826250001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 250, dap: 105, npk: 70, ssp: 125 },
                remarks: 'आधार कार्ड व भू-अभिलेख के साथ संपर्क करें।',
                lastUpdated: 'आज, 08:30 AM'
            },
            {
                id: 'mp_sat_01',
                name: 'सतना व मैहर सहकारी समिति',
                district: 'Satna',
                districtHi: 'सतना / मैहर',
                tehsil: 'सतना / मैहर',
                type: 'प्राथमिक कृषि साख सहकारी संस्था (PACS)',
                address: 'रीवा रोड, सतना, मध्य प्रदेश',
                phone: '9826260001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 225, dap: 95, npk: 65, ssp: 115 },
                remarks: 'खाद वितरण जारी है।',
                lastUpdated: 'कल, 05:00 PM'
            },
            {
                id: 'mp_kha_01',
                name: 'खंडवा व खरगोन सहकारी समिति',
                district: 'Khargone',
                districtHi: 'खंडवा / खरगोन',
                tehsil: 'खरगोन / बड़वाह',
                type: 'सहकारी विपणन संघ',
                address: 'मंडी रोड, खरगोन, मध्य प्रदेश',
                phone: '9827270001',
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: { urea: 305, dap: 135, npk: 100, ssp: 155 },
                remarks: 'यूरिया, डीएपी व सुपर खाद उपलब्ध।',
                lastUpdated: 'आज, 09:02 AM'
            }
        ];

        let centersData = [];
        let favoriteSocietyIds = ['mp_raj_chatukheda'];
        let showOnlyFavorites = false;
        let currentlySelectedCenterForModal = null;
        window.DEFAULT_MP_CENTERS = DEFAULT_MP_CENTERS;
        window.centersData = centersData;

        document.addEventListener('DOMContentLoaded', () => {
            loadDataFromStorage();
            loadFavoritesAndTokens();
            updateDashboardStats();
            applyFilters();
            calculateFertilizerNeed();
            lucide.createIcons();
        });

        function loadDataFromStorage() {
            try {
                const raw = localStorage.getItem(STORAGE_KEY);
                if (raw) {
                    const parsed = JSON.parse(raw);
                    if (Array.isArray(parsed) && parsed.length > 0) {
                        centersData = parsed;
                        window.centersData = centersData;
                        return;
                    }
                }
            } catch (err) {
                console.error('Error reading localStorage:', err);
            }
            centersData = JSON.parse(JSON.stringify(DEFAULT_MP_CENTERS));
            window.centersData = centersData;
            try {
                localStorage.setItem(STORAGE_KEY, JSON.stringify(centersData));
            } catch (err) {}
        }

        async function saveDataToStorage() {
            window.centersData = centersData;
            try {
                localStorage.setItem(STORAGE_KEY, JSON.stringify(centersData));
            } catch (err) {
                console.error('Error saving to localStorage:', err);
            }
            if (typeof window.persistCentersToCloud === 'function') {
                return await window.persistCentersToCloud(centersData);
            }
            return false;
        }

        function loadFavoritesAndTokens() {
            try {
                const savedFavs = localStorage.getItem(FAV_STORAGE_KEY);
                if (savedFavs) favoriteSocietyIds = JSON.parse(savedFavs);
            } catch (e) {
                console.error(e);
            }
            updateBadgesCount();
        }

        function updateBadgesCount() {
            const favEl = document.getElementById('favCountBadge');
            if (favEl) favEl.textContent = favoriteSocietyIds.length;
        }

        function getCenterStatus(stock) {
            const total = (stock.urea || 0) + (stock.dap || 0) + (stock.npk || 0) + (stock.ssp || 0);
            if (total === 0) {
                return {
                    code: 'OUT',
                    label: 'स्टॉक समाप्त',
                    badgeClass: 'bg-rose-50 text-rose-700 border-rose-200',
                    dotClass: 'bg-rose-500'
                };
            }
            if (total < 150 || stock.urea < 50 || stock.dap < 40) {
                return {
                    code: 'LOW',
                    label: 'कम स्टॉक',
                    badgeClass: 'bg-amber-50 text-amber-800 border-amber-200',
                    dotClass: 'bg-amber-500'
                };
            }
            return {
                code: 'AVAILABLE',
                label: 'पर्याप्त उपलब्ध',
                badgeClass: 'bg-agri-50 text-agri-800 border-agri-200',
                dotClass: 'bg-agri-600'
            };
        }

        function updateDashboardStats() {
            if (Array.isArray(window.centersData) && window.centersData.length > 0) {
                centersData = window.centersData;
            }
            let totalUrea = 0;
            let totalDap = 0;
            let lowOrOutCount = 0;

            centersData.forEach(c => {
                totalUrea += Number(c.stock.urea || 0);
                totalDap += Number(c.stock.dap || 0);
                const st = getCenterStatus(c.stock);
                if (st.code === 'LOW' || st.code === 'OUT') {
                    lowOrOutCount++;
                }
            });

            document.getElementById('statTotalCenters').textContent = centersData.length;
            document.getElementById('statTotalUrea').textContent = totalUrea.toLocaleString('en-IN');
            document.getElementById('statTotalDap').textContent = totalDap.toLocaleString('en-IN');
            document.getElementById('statLowOrOut').textContent = lowOrOutCount;
        }

        function applyFilters() {
            if (Array.isArray(window.centersData) && window.centersData.length > 0) {
                centersData = window.centersData;
            }
            const searchText = document.getElementById('searchInput').value.trim().toLowerCase();
            const districtVal = document.getElementById('districtFilter').value;
            const fertilizerVal = document.getElementById('fertilizerFilter').value;
            const statusVal = document.getElementById('statusFilter').value;

            const filtered = centersData.filter(center => {
                if (showOnlyFavorites && !favoriteSocietyIds.includes(center.id)) {
                    return false;
                }

                const matchesSearch =
                    center.name.toLowerCase().includes(searchText) ||
                    center.districtHi.toLowerCase().includes(searchText) ||
                    center.district.toLowerCase().includes(searchText) ||
                    center.tehsil.toLowerCase().includes(searchText) ||
                    center.address.toLowerCase().includes(searchText);

                const matchesDistrict = (districtVal === 'ALL') || (center.district === districtVal);
                const matchesFertilizer = (fertilizerVal === 'ALL') || (center.stock[fertilizerVal] > 0);
                const statusObj = getCenterStatus(center.stock);
                const matchesStatus = (statusVal === 'ALL') || (statusObj.code === statusVal);

                return matchesSearch && matchesDistrict && matchesFertilizer && matchesStatus;
            });

            filtered.sort((a, b) => {
                const aFav = favoriteSocietyIds.includes(a.id) ? 1 : 0;
                const bFav = favoriteSocietyIds.includes(b.id) ? 1 : 0;
                return bFav - aFav;
            });

            renderCentersGrid(filtered, fertilizerVal);
        }

        window.updateDashboardStats = updateDashboardStats;
        window.applyFilters = applyFilters;
        window.updateCloudStatusUI = function(isConnected) {
            const badge = document.getElementById('cloudSyncBadge');
            const dot = document.getElementById('cloudSyncDot');
            const text = document.getElementById('cloudSyncText');
            if (!badge || !dot || !text) return;
            if (isConnected) {
                badge.className = 'inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-semibold bg-agri-100 text-agri-800 border border-agri-200';
                dot.className = 'w-1.5 h-1.5 rounded-full bg-agri-600 animate-pulse';
                text.textContent = 'CLOUD SAVED';
            } else {
                badge.className = 'inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-semibold bg-amber-100 text-amber-800 border border-amber-200';
                dot.className = 'w-1.5 h-1.5 rounded-full bg-amber-500';
                text.textContent = 'LOCAL MODE';
            }
        };

        function toggleFavoriteSociety(centerId) {
            const idx = favoriteSocietyIds.indexOf(centerId);
            if (idx > -1) {
                favoriteSocietyIds.splice(idx, 1);
                showToast('पसंदीदा सूची से हटाया गया');
            } else {
                favoriteSocietyIds.push(centerId);
                showToast('⭐ मेरी पसंदीदा सोसायटी में पिन किया गया!');
            }
            try {
                localStorage.setItem(FAV_STORAGE_KEY, JSON.stringify(favoriteSocietyIds));
            } catch (e) {}
            updateBadgesCount();
            applyFilters();
        }

        function toggleFavoritesFilter() {
            showOnlyFavorites = !showOnlyFavorites;
            const btn = document.getElementById('favFilterToggleBtn');
            if (showOnlyFavorites) {
                btn.classList.add('ring-2', 'ring-amber-500', 'bg-amber-200');
            } else {
                btn.classList.remove('ring-2', 'ring-amber-500', 'bg-amber-200');
            }
            applyFilters();
        }

        function resetFilters() {
            document.getElementById('searchInput').value = '';
            document.getElementById('districtFilter').value = 'ALL';
            document.getElementById('fertilizerFilter').value = 'ALL';
            document.getElementById('statusFilter').value = 'ALL';
            showOnlyFavorites = false;
            const btn = document.getElementById('favFilterToggleBtn');
            if (btn) btn.classList.remove('ring-2', 'ring-amber-500', 'bg-amber-200');
            applyFilters();
        }

        function renderCentersGrid(centersList, highlightedFert = 'ALL') {
            const grid = document.getElementById('centersGrid');
            const emptyState = document.getElementById('emptyState');
            const countBadge = document.getElementById('resultCountBadge');

            countBadge.textContent = `${centersList.length} समितियाँ दिख रही हैं`;

            if (centersList.length === 0) {
                grid.innerHTML = '';
                emptyState.classList.remove('hidden');
                return;
            }

            emptyState.classList.add('hidden');

            grid.innerHTML = centersList.map(center => {
                const status = getCenterStatus(center.stock);
                const isFeaturedChatukheda = center.id === 'mp_raj_chatukheda';
                const isFav = favoriteSocietyIds.includes(center.id);

                const getFertBoxStyle = (fertKey, qty) => {
                    const isHighlighted = highlightedFert === fertKey;
                    if (qty === 0) {
                        return 'bg-slate-50 border-slate-200 opacity-60';
                    }
                    if (isHighlighted) {
                        return 'bg-agri-50 border-agri-500 ring-2 ring-agri-500/20';
                    }
                    return 'bg-slate-50/70 border-slate-200/80 hover:border-slate-300';
                };

                return `
                <article class="bg-white rounded-2xl border ${isFav ? 'border-amber-400 ring-2 ring-amber-400/20' : (isFeaturedChatukheda ? 'border-agri-500 ring-2 ring-agri-500/20' : 'border-slate-200/90')} shadow-sm hover:shadow-lg transition-all duration-200 flex flex-col justify-between overflow-hidden group">
                    <div class="p-5 space-y-4">
                        <div class="flex flex-wrap items-center justify-between gap-2">
                            <span class="inline-flex items-center gap-1.5 px-2.5 py-1 rounded-lg text-xs font-bold bg-slate-100 text-slate-700">
                                <i data-lucide="map-pin" class="w-3.5 h-3.5 text-saffron-600 shrink-0"></i>
                                जिला: ${center.districtHi} • ${center.tehsil}
                            </span>
                            <div class="flex items-center gap-1.5">
                                <button onclick="toggleFavoriteSociety('${center.id}')" title="${isFav ? 'पिन हटाएं' : 'मेरी सोसायटी पिन करें'}" class="w-7 h-7 rounded-full border ${isFav ? 'bg-amber-50 border-amber-300 text-amber-500' : 'bg-slate-50 border-slate-200 text-slate-400 hover:text-amber-500'} flex items-center justify-center transition-colors cursor-pointer">
                                    <i data-lucide="star" class="w-3.5 h-3.5 ${isFav ? 'fill-amber-400' : ''}"></i>
                                </button>
                                <span class="inline-flex items-center gap-1.5 px-2.5 py-1 rounded-full text-xs font-bold border ${status.badgeClass}">
                                    <span class="w-2 h-2 rounded-full ${status.dotClass}"></span>
                                    ${status.label}
                                </span>
                            </div>
                        </div>

                        <div>
                            <div class="flex items-center gap-2 flex-wrap">
                                <h3 class="text-lg font-extrabold text-slate-900 group-hover:text-agri-800 transition-colors leading-snug">
                                    ${center.name}
                                </h3>
                                ${isFeaturedChatukheda ? '<span class="px-2 py-0.5 text-[11px] font-bold rounded-md bg-agri-100 text-agri-800">मुख्य केंद्र</span>' : ''}
                            </div>
                            <p class="text-xs font-medium text-agri-700 mt-0.5">${center.type}</p>
                            <p class="text-xs text-slate-500 mt-1 flex items-start gap-1">
                                <i data-lucide="navigation" class="w-3.5 h-3.5 text-slate-400 shrink-0 mt-0.5"></i>
                                <span>${center.address}</span>
                            </p>
                        </div>

                        <div class="grid grid-cols-2 gap-2.5 pt-1">
                            <div class="p-3 rounded-xl border ${getFertBoxStyle('urea', center.stock.urea)} transition-colors">
                                <div class="flex items-center justify-between">
                                    <span class="text-xs font-bold text-blue-900">यूरिया (Urea)</span>
                                    <span class="w-2 h-2 rounded-full ${center.stock.urea > 0 ? 'bg-blue-500' : 'bg-slate-300'}"></span>
                                </div>
                                <p class="text-xl font-extrabold text-slate-900 mt-1">
                                    ${center.stock.urea} <span class="text-xs font-semibold text-slate-500">कट्टे</span>
                                </p>
                            </div>

                            <div class="p-3 rounded-xl border ${getFertBoxStyle('dap', center.stock.dap)} transition-colors">
                                <div class="flex items-center justify-between">
                                    <span class="text-xs font-bold text-emerald-900">डीएपी (DAP)</span>
                                    <span class="w-2 h-2 rounded-full ${center.stock.dap > 0 ? 'bg-emerald-500' : 'bg-slate-300'}"></span>
                                </div>
                                <p class="text-xl font-extrabold text-slate-900 mt-1">
                                    ${center.stock.dap} <span class="text-xs font-semibold text-slate-500">कट्टे</span>
                                </p>
                            </div>

                            <div class="p-3 rounded-xl border ${getFertBoxStyle('npk', center.stock.npk)} transition-colors">
                                <div class="flex items-center justify-between">
                                    <span class="text-xs font-bold text-amber-900">एनपीके (NPK)</span>
                                    <span class="w-2 h-2 rounded-full ${center.stock.npk > 0 ? 'bg-amber-500' : 'bg-slate-300'}"></span>
                                </div>
                                <p class="text-xl font-extrabold text-slate-900 mt-1">
                                    ${center.stock.npk} <span class="text-xs font-semibold text-slate-500">कट्टे</span>
                                </p>
                            </div>

                            <div class="p-3 rounded-xl border ${getFertBoxStyle('ssp', center.stock.ssp)} transition-colors">
                                <div class="flex items-center justify-between">
                                    <span class="text-xs font-bold text-purple-900">सुपर (SSP)</span>
                                    <span class="w-2 h-2 rounded-full ${center.stock.ssp > 0 ? 'bg-purple-500' : 'bg-slate-300'}"></span>
                                </div>
                                <p class="text-xl font-extrabold text-slate-900 mt-1">
                                    ${center.stock.ssp} <span class="text-xs font-semibold text-slate-500">कट्टे</span>
                                </p>
                            </div>
                        </div>

                        ${center.remarks ? `
                        <div class="bg-amber-50/70 border border-amber-200/70 rounded-xl px-3 py-2 text-xs text-amber-900 flex items-start gap-2">
                            <i data-lucide="megaphone" class="w-3.5 h-3.5 text-amber-600 shrink-0 mt-0.5"></i>
                            <span>${center.remarks}</span>
                        </div>` : ''}
                    </div>

                    <div class="px-4 py-3 bg-slate-50 border-t border-slate-100 flex flex-wrap items-center justify-between gap-2">
                        <div class="flex items-center gap-1.5 text-[11px] text-slate-500">
                            <i data-lucide="clock" class="w-3.5 h-3.5 text-slate-400"></i>
                            <span>अपडेट: <strong class="text-slate-700 font-semibold">${center.lastUpdated}</strong></span>
                        </div>
                        <div class="flex items-center gap-1.5 ml-auto">
                            <button onclick="openDetailsModal('${center.id}')" class="inline-flex items-center gap-1 px-3 py-1.5 rounded-xl text-xs font-bold text-slate-700 bg-white border border-slate-300 hover:bg-slate-100 transition-colors cursor-pointer" title="केंद्र का पता व संपर्क देखें">
                                <i data-lucide="info" class="w-3.5 h-3.5 text-agri-600"></i>
                                <span>विवरण</span>
                            </button>
                        </div>
                    </div>
                </article>
                `;
            }).join('');

            lucide.createIcons();
        }

        function openDetailsModal(centerId) {
            const center = centersData.find(c => c.id === centerId);
            if (!center) return;
            currentlySelectedCenterForModal = center;

            document.getElementById('modalDetailTitle').textContent = center.name;
            document.getElementById('modalDetailBody').innerHTML = `
                <div class="space-y-3">
                    <div class="flex items-center justify-between bg-slate-50 p-3 rounded-xl border border-slate-200/80">
                        <span class="text-slate-500 font-medium">जिला व तहसील</span>
                        <span class="font-bold text-slate-800">जिला: ${center.districtHi} • ${center.tehsil}</span>
                    </div>
                    <div class="flex items-start justify-between gap-4 bg-slate-50 p-3 rounded-xl border border-slate-200/80">
                        <span class="text-slate-500 font-medium shrink-0">पता</span>
                        <span class="font-semibold text-slate-800 text-right">${center.address}</span>
                    </div>
                    <div class="flex items-center justify-between bg-slate-50 p-3 rounded-xl border border-slate-200/80">
                        <span class="text-slate-500 font-medium">वितरण समय</span>
                        <span class="font-semibold text-slate-800">${center.timings || 'सुबह 10:00 से शाम 05:00'}</span>
                    </div>
                    <div class="flex items-center justify-between bg-slate-50 p-3 rounded-xl border border-slate-200/80">
                        <span class="text-slate-500 font-medium">संपर्क नंबर</span>
                        <a href="tel:${center.phone || ''}" class="font-bold text-agri-700 hover:underline">${center.phone || 'उपलब्ध नहीं'}</a>
                    </div>

                    <div class="pt-2">
                        <p class="text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">वर्तमान खाद स्टॉक (कट्टों में)</p>
                        <div class="grid grid-cols-4 gap-2 text-center">
                            <div class="p-2.5 rounded-xl bg-blue-50 border border-blue-100">
                                <span class="text-[11px] font-bold text-blue-800 block">यूरिया</span>
                                <span class="text-base font-extrabold text-slate-900">${center.stock.urea}</span>
                            </div>
                            <div class="p-2.5 rounded-xl bg-emerald-50 border border-emerald-100">
                                <span class="text-[11px] font-bold text-emerald-800 block">डीएपी</span>
                                <span class="text-base font-extrabold text-slate-900">${center.stock.dap}</span>
                            </div>
                            <div class="p-2.5 rounded-xl bg-amber-50 border border-amber-100">
                                <span class="text-[11px] font-bold text-amber-800 block">NPK</span>
                                <span class="text-base font-extrabold text-slate-900">${center.stock.npk}</span>
                            </div>
                            <div class="p-2.5 rounded-xl bg-purple-50 border border-purple-100">
                                <span class="text-[11px] font-bold text-purple-800 block">SSP</span>
                                <span class="text-base font-extrabold text-slate-900">${center.stock.ssp}</span>
                            </div>
                        </div>
                    </div>

                    ${center.remarks ? `
                    <div class="p-3 rounded-xl bg-amber-50 border border-amber-200 text-xs text-amber-900">
                        <strong>सूचना:</strong> ${center.remarks}
                    </div>` : ''}
                </div>
            `;

            document.getElementById('detailsModal').classList.remove('hidden');
            lucide.createIcons();
        }

        function closeDetailsModal() {
            document.getElementById('detailsModal').classList.add('hidden');
            currentlySelectedCenterForModal = null;
        }

        function copyCenterShareText() {
            if (!currentlySelectedCenterForModal) return;
            const c = currentlySelectedCenterForModal;
            const shareText = `🌾 *X2A Khadya Stock Checker (MP)* 🌾\n📍 *${c.name}* (जिला: ${c.districtHi}, ${c.tehsil})\n📌 पता: ${c.address}\n\n📦 *उपलब्ध खाद स्टॉक:*\n• यूरिया (Urea): *${c.stock.urea} कट्टे*\n• डीएपी (DAP): *${c.stock.dap} कट्टे*\n• एनपीके (NPK): *${c.stock.npk} कट्टे*\n• सुपर (SSP): *${c.stock.ssp} कट्टे*\n\n🕒 अपडेट: ${c.lastUpdated}\n📺 संचालक: MR.INDIAN X2A (YT CHANNEL)`;

            const textArea = document.createElement('textarea');
            textArea.value = shareText;
            document.body.appendChild(textArea);
            textArea.select();
            try {
                document.execCommand('copy');
                showToast('WhatsApp पर शेयर करने के लिए जानकारी कॉपी हो गई!');
            } catch (err) {
                showToast('कॉपी करने में समस्या आई।');
            }
            document.body.removeChild(textArea);
        }

        function openAdminModal() {
            const modal = document.getElementById('adminModal');
            const authForm = document.getElementById('adminAuthForm');
            const tabsBar = document.getElementById('adminTabsBar');
            const formUpdate = document.getElementById('adminUpdateForm');
            const formAdd = document.getElementById('adminAddForm');
            const passInput = document.getElementById('adminPasswordInput');
            const errMsg = document.getElementById('adminAuthError');

            modal.classList.remove('hidden');

            if (!isAdminUnlocked) {
                authForm.classList.remove('hidden');
                tabsBar.classList.add('hidden');
                formUpdate.classList.add('hidden');
                formAdd.classList.add('hidden');
                errMsg.classList.add('hidden');
                passInput.value = '';
                setTimeout(() => passInput.focus(), 50);
                lucide.createIcons();
                return;
            }

            unlockAdminPanelUI();
        }

        function unlockAdminPanelUI() {
            document.getElementById('adminAuthForm').classList.add('hidden');
            document.getElementById('adminTabsBar').classList.remove('hidden');

            const distSelect = document.getElementById('adminDistrictFilter');
            const uniqueDistricts = [...new Set(centersData.map(c => c.district))];
            distSelect.innerHTML = `<option value="ALL">सभी जिले (${centersData.length})</option>` +
                uniqueDistricts.map(d => `<option value="${d}">${DISTRICT_HINDI_MAP[d] || d} (${d})</option>`).join('');

            filterAdminCentersByDistrict();
            switchAdminTab('update');
            lucide.createIcons();
        }

        function handleAdminPasswordSubmit(e) {
            e.preventDefault();
            const enteredPass = document.getElementById('adminPasswordInput').value;
            const errMsg = document.getElementById('adminAuthError');

            if (enteredPass === ADMIN_PASSCODE) {
                isAdminUnlocked = true;
                errMsg.classList.add('hidden');
                unlockAdminPanelUI();
                showToast('🔓 Admin पैनल अनलॉक हो गया!');
            } else {
                errMsg.classList.remove('hidden');
                showToast('गलत पासवर्ड! कृपया सही पासवर्ड डालें।');
            }
        }

        function toggleAdminPasswordVisibility() {
            const input = document.getElementById('adminPasswordInput');
            const icon = document.getElementById('adminPassEyeIcon');
            if (input.type === 'password') {
                input.type = 'text';
                icon.setAttribute('data-lucide', 'eye-off');
            } else {
                input.type = 'password';
                icon.setAttribute('data-lucide', 'eye');
            }
            lucide.createIcons();
        }

        function filterAdminCentersByDistrict() {
            const dist = document.getElementById('adminDistrictFilter').value;
            const select = document.getElementById('adminCenterSelect');
            const filtered = dist === 'ALL' ? centersData : centersData.filter(c => c.district === dist);

            select.innerHTML = filtered.map(c =>
                `<option value="${c.id}">[${c.districtHi}] ${c.name}</option>`
            ).join('');

            populateAdminForm();
        }

        function closeAdminModal() {
            document.getElementById('adminModal').classList.add('hidden');
            isAdminUnlocked = false;
        }

        function switchAdminTab(tab) {
            const tabUpdate = document.getElementById('tabUpdateStock');
            const tabAdd = document.getElementById('tabAddCenter');
            const formUpdate = document.getElementById('adminUpdateForm');
            const formAdd = document.getElementById('adminAddForm');

            if (tab === 'update') {
                tabUpdate.classList.add('border-agri-600', 'text-agri-700');
                tabUpdate.classList.remove('border-transparent', 'text-slate-500');
                tabAdd.classList.remove('border-agri-600', 'text-agri-700');
                tabAdd.classList.add('border-transparent', 'text-slate-500');
                formUpdate.classList.remove('hidden');
                formAdd.classList.add('hidden');
            } else {
                tabAdd.classList.add('border-agri-600', 'text-agri-700');
                tabAdd.classList.remove('border-transparent', 'text-slate-500');
                tabUpdate.classList.remove('border-agri-600', 'text-agri-700');
                tabUpdate.classList.add('border-transparent', 'text-slate-500');
                formAdd.classList.remove('hidden');
                formUpdate.classList.add('hidden');
            }
        }

        function populateAdminForm() {
            const select = document.getElementById('adminCenterSelect');
            const center = centersData.find(c => c.id === select.value);
            if (!center) return;

            document.getElementById('editUrea').value = center.stock.urea;
            document.getElementById('editDap').value = center.stock.dap;
            document.getElementById('editNpk').value = center.stock.npk;
            document.getElementById('editSsp').value = center.stock.ssp;
            document.getElementById('editRemarks').value = center.remarks || '';
        }
        async function handleUpdateStockSubmit(e) {
            e.preventDefault();
            const centerId = document.getElementById('adminCenterSelect').value;
            const idx = centersData.findIndex(c => c.id === centerId);
            if (idx === -1) return;

            centersData[idx].stock = {
                urea: Math.max(0, parseInt(document.getElementById('editUrea').value, 10) || 0),
                dap: Math.max(0, parseInt(document.getElementById('editDap').value, 10) || 0),
                npk: Math.max(0, parseInt(document.getElementById('editNpk').value, 10) || 0),
                ssp: Math.max(0, parseInt(document.getElementById('editSsp').value, 10) || 0),
            };
            centersData[idx].remarks = document.getElementById('editRemarks').value.trim();
            centersData[idx].lastUpdated = getFormattedNowTime();

            updateDashboardStats();
            applyFilters();
            const savedToCloud = await saveDataToStorage();
            closeAdminModal();
            if (savedToCloud) {
                showToast(`✅ ${centersData[idx].name} का नया स्टॉक क्लाउड पर स्थायी रूप से सेव हो गया!`);
            } else {
                showToast(`✅ ${centersData[idx].name} का स्टॉक अपडेट हो गया!`);
            }
        }

        async function handleAddCenterSubmit(e) {
            e.preventDefault();
            const name = document.getElementById('newCenterName').value.trim();
            const district = document.getElementById('newCenterDistrict').value;
            const districtHi = DISTRICT_HINDI_MAP[district] || district;
            const tehsil = document.getElementById('newCenterTehsil').value.trim();
            const phone = document.getElementById('newCenterPhone').value.trim();
            const address = document.getElementById('newCenterAddress').value.trim();

            const newCenter = {
                id: 'mp_custom_' + Date.now(),
                name,
                district,
                districtHi,
                tehsil,
                type: 'सहकारी समिति केंद्र (MP)',
                address,
                phone,
                timings: 'सोम - शनि, सुबह 10:00 से शाम 05:00',
                stock: {
                    urea: Math.max(0, parseInt(document.getElementById('newUrea').value, 10) || 0),
                    dap: Math.max(0, parseInt(document.getElementById('newDap').value, 10) || 0),
                    npk: Math.max(0, parseInt(document.getElementById('newNpk').value, 10) || 0),
                    ssp: Math.max(0, parseInt(document.getElementById('newSsp').value, 10) || 0),
                },
                remarks: 'नया केंद्र जोड़ा गया।',
                lastUpdated: getFormattedNowTime()
            };

            centersData.unshift(newCenter);
            updateDashboardStats();
            applyFilters();
            document.getElementById('adminAddForm').reset();
            closeAdminModal();
            await saveDataToStorage();
            showToast(`✅ नई समिति "${name}" स्थायी रूप से जोड़ दी गई!`);
        }

        async function resetDemoData() {
            centersData = JSON.parse(JSON.stringify(DEFAULT_MP_CENTERS));
            updateDashboardStats();
            applyFilters();
            filterAdminCentersByDistrict();
            await saveDataToStorage();
            showToast('मध्य प्रदेश की सभी समितियों का डिफ़ॉल्ट स्टॉक रीसेट हो गया!');
        }

        function getFormattedNowTime() {
            const now = new Date();
            const timeStr = now.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' });
            return `आज, ${timeStr}`;
        }

        let toastTimer = null;
        function showToast(message) {
            const toast = document.getElementById('toastNotification');
            const msgEl = document.getElementById('toastMessage');
            msgEl.textContent = message;

            toast.classList.remove('translate-y-20', 'opacity-0');
            toast.classList.add('translate-y-0', 'opacity-100');

            clearTimeout(toastTimer);
            toastTimer = setTimeout(() => {
                toast.classList.remove('translate-y-0', 'opacity-100');
                toast.classList.add('translate-y-20', 'opacity-0');
            }, 3000);
        }

        document.addEventListener('keydown', (e) => {
            if (e.key === 'Escape') {
                closeDetailsModal();
                closeAdminModal();
                closeCalculatorModal();
            }
        });

        function openCalculatorModal() {
            calculateFertilizerNeed();
            document.getElementById('calculatorModal').classList.remove('hidden');
            lucide.createIcons();
        }

        function closeCalculatorModal() {
            document.getElementById('calculatorModal').classList.add('hidden');
        }

        function calculateFertilizerNeed() {
            const area = Math.max(0.1, parseFloat(document.getElementById('calcLandArea').value) || 1);
            const unit = document.getElementById('calcLandUnit').value;
            const crop = document.getElementById('calcCropType').value;

            let bighaMultiplier = 1;
            if (unit === 'acre') bighaMultiplier = 1.6;
            if (unit === 'hectare') bighaMultiplier = 4.0;
            const totalBigha = area * bighaMultiplier;

            const cropNorms = {
                wheat: { urea: 1.5, dap: 0.8, label: 'गेहूँ (Wheat)' },
                gram: { urea: 0.4, dap: 0.8, label: 'चना / मसूर (Gram)' },
                soybean: { urea: 0.5, dap: 0.9, label: 'सोयाबीन / मक्का' },
                garlic: { urea: 1.8, dap: 1.2, label: 'लहसुन / प्याज' }
            };

            const norm = cropNorms[crop] || cropNorms.wheat;
            const reqUrea = Math.max(1, Math.ceil(totalBigha * norm.urea));
            const reqDap = Math.max(1, Math.ceil(totalBigha * norm.dap));
            const estTotal = Math.round((reqUrea * 266.5) + (reqDap * 1350));

            document.getElementById('calculatorResultBox').innerHTML = `
                <div class="flex items-center justify-between text-xs text-slate-500 border-b border-slate-100 pb-2">
                    <span>फसल: <strong class="text-slate-800">${norm.label}</strong></span>
                    <span>अनुमानित रकबा: <strong class="text-slate-800">${totalBigha.toFixed(1)} बीघा</strong></span>
                </div>
                <div class="grid grid-cols-3 gap-2 pt-1 text-center">
                    <div class="p-2 rounded-xl bg-blue-50">
                        <span class="text-[11px] font-bold text-blue-800 block">आवश्यक यूरिया</span>
                        <span class="text-base font-extrabold text-slate-900">${reqUrea} कट्टे</span>
                    </div>
                    <div class="p-2 rounded-xl bg-emerald-50">
                        <span class="text-[11px] font-bold text-emerald-800 block">आवश्यक डीएपी</span>
                        <span class="text-base font-extrabold text-slate-900">${reqDap} कट्टे</span>
                    </div>
                    <div class="p-2 rounded-xl bg-agri-50">
                        <span class="text-[11px] font-bold text-agri-800 block">कुल सरकारी खर्च</span>
                        <span class="text-base font-extrabold text-agri-800">₹${estTotal.toLocaleString('en-IN')}</span>
                    </div>
                </div>
            `;
        }

        function startVoiceSearch() {
            const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
            if (!SpeechRecognition) {
                showToast('आपके ब्राउज़र में वॉइस सर्च उपलब्ध नहीं है। कृपया टाइप करके खोजें।');
                return;
            }
            const recognition = new SpeechRecognition();
            recognition.lang = 'hi-IN';
            recognition.interimResults = false;

            const micBtn = document.getElementById('voiceSearchBtn');
            micBtn.classList.add('bg-rose-500', 'text-white', 'animate-pulse');
            showToast('🎤 बोलिए... (जैसे: चटुखेड़ा, राजगढ़, इंदौर)');

            recognition.onresult = (event) => {
                const spokenText = event.results[0][0].transcript;
                document.getElementById('searchInput').value = spokenText;
                applyFilters();
                showToast(`आपने बोला: "${spokenText}"`);
            };

            recognition.onerror = () => {
                showToast('आवाज स्पष्ट नहीं सुनाई दी, कृपया पुनः प्रयास करें।');
            };

            recognition.onend = () => {
                micBtn.classList.remove('bg-rose-500', 'text-white', 'animate-pulse');
            };

            recognition.start();
        }

        function toggleSunlightMode() {
            document.body.classList.toggle('sunlight-mode');
            const isSunlight = document.body.classList.contains('sunlight-mode');
            showToast(isSunlight ? '☀️ धूप मोड (High-Contrast) चालू किया गया!' : 'सामान्य मोड चालू किया गया।');
        }
    </script>
</body>
</html>
