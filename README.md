<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>클린한사람들</title>
  <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;600;700;800;900&display=swap" rel="stylesheet" />
  <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <style>
    * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
    body { margin: 0; padding: 0; background: #f2f5f3; font-family: 'Noto Sans KR', sans-serif; }
    input, textarea, select, button { font-family: 'Noto Sans KR', sans-serif; }
  </style>
</head>
<body>
<div id="root"></div>

<script type="text/babel">
const { useState, useEffect, useRef } = React;

// ============================================================
// ⚙️  설정
// ============================================================
const ADMIN        = { name: "관리자", pin: "9999", role: "admin" };
const STORAGE_KEY  = "cleaning-reports-v2";
const LEADERS_KEY  = "cleaning-leaders-v2";
const GAS_URL      = "https://script.google.com/macros/s/AKfycbxOaNdRxosciGoqz84j2LSY21UuxWrJCCsb4rMAI5s_uODjcwqpVJJLmzWgBc4uXDfepA/exec";
const ADMIN_EMAIL  = "kjg2000914@naver.com";

const DEFAULT_LEADERS = [
  { name: "김팀장", pin: "1111" },
  { name: "이팀장", pin: "2222" },
  { name: "박팀장", pin: "3333" },
  { name: "최팀장", pin: "4444" },
  { name: "정팀장", pin: "5555" },
];

// ============================================================
// 🗄️  스토리지 (localStorage 기반 — 기기 로컬 저장)
//     팀원들이 같은 데이터를 공유하려면 서버가 필요합니다.
//     현재는 각 기기별로 데이터가 저장됩니다.
// ============================================================
const db = {
  get: (key) => { try { const v = localStorage.getItem(key); return v ? JSON.parse(v) : null; } catch { return null; } },
  set: (key, val) => { try { localStorage.setItem(key, JSON.stringify(val)); } catch(e) { alert("저장 오류: " + e.message); } },
};

// ============================================================
// 유틸
// ============================================================
const todayStr = () => new Date().toISOString().slice(0, 10);
const fmtKRW   = (n) => Number(n || 0).toLocaleString("ko-KR") + "원";
const monthOf  = (d) => d?.slice(0, 7) ?? "";
const GREEN    = "#0f4c2a";
const LGREEN   = "#1a8048";

// ============================================================
// 🏠  App
// ============================================================
function App() {
  const [user,    setUser]    = useState(null);
  const [reports, setReports] = useState([]);
  const [leaders, setLeaders] = useState([]);
  const [screen,  setScreen]  = useState("login");

  useEffect(() => { loadAll(); }, []);

  const loadAll = () => {
    const r = db.get(STORAGE_KEY);
    if (r) setReports(r);

    const l = db.get(LEADERS_KEY);
    if (l) setLeaders(l);
    else { db.set(LEADERS_KEY, DEFAULT_LEADERS); setLeaders(DEFAULT_LEADERS); }
  };

  const saveReports = (updated) => { setReports(updated); db.set(STORAGE_KEY, updated); };
  const saveLeaders = (updated) => { setLeaders(updated); db.set(LEADERS_KEY, updated); };
  const deleteReport = (id) => saveReports(reports.filter(r => r.id !== id));
  const editReport   = (u)  => saveReports(reports.map(r => r.id === u.id ? u : r));

  const login  = (u) => { setUser(u); setScreen(u.role === "admin" ? "admin" : "home"); };
  const logout = () => { setUser(null); setScreen("login"); };

  return (
    <div style={s.app}>
      {screen === "login" && <LoginScreen onLogin={login} leaders={leaders} />}
      {screen !== "login" && <>
        <Header user={user} onLogout={logout} screen={screen} setScreen={setScreen} />
        <div style={s.content}>
          {screen === "home"      && <HomeScreen    user={user} reports={reports} setScreen={setScreen} onDelete={deleteReport} onEdit={editReport} />}
          {screen === "submit"    && <SubmitScreen  user={user} reports={reports} saveReports={saveReports} />}
          {screen === "myreports" && <MyReportsScreen user={user} reports={reports} onDelete={deleteReport} onEdit={editReport} />}
          {screen === "admin"     && <AdminScreen   reports={reports} leaders={leaders} loadAll={loadAll} onDelete={deleteReport} onEdit={editReport} />}
          {screen === "stats"     && <StatsScreen   reports={reports} user={user} leaders={leaders} />}
          {screen === "members"   && <MembersScreen leaders={leaders} saveLeaders={saveLeaders} />}
          {screen === "search"    && <SearchScreen  reports={reports} user={user} onDelete={deleteReport} onEdit={editReport} />}
        </div>
        <BottomNav role={user?.role} screen={screen} setScreen={setScreen} />
      </>}
    </div>
  );
}

// ============================================================
// 🔐  Login
// ============================================================
function LoginScreen({ onLogin, leaders }) {
  const [selected, setSelected] = useState(null);
  const [pin, setPin] = useState("");
  const [error, setError] = useState(false);

  const handlePinInput = (digit) => {
    if (pin.length >= 4) return;
    const next = pin + digit;
    setPin(next);
    setError(false);
    if (next.length === 4) {
      setTimeout(() => {
        if (selected.pin === next) onLogin(selected);
        else { setError(true); setPin(""); }
      }, 200);
    }
  };

  return (
    <div style={s.loginBg}>
      <div style={s.loginCard}>
        <div style={s.logoArea}>
          <div style={{ fontSize:56 }}>🧹</div>
          <h1 style={s.logoTitle}>클린한사람들</h1>
          <p style={s.logoSub}>현장 정산 관리 시스템</p>
        </div>

        {!selected ? (<>
          <p style={s.selectHint}>이름을 선택하세요</p>
          <p style={s.groupLabel}>팀장</p>
          <div style={s.nameGrid}>
            {leaders.map(u => (
              <button key={u.name} style={s.nameBtn} onClick={() => setSelected({...u, role:"leader"})}>{u.name}</button>
            ))}
          </div>
          <p style={s.groupLabel}>관리자</p>
          <div style={s.nameGrid}>
            <button style={{...s.nameBtn,...s.adminNameBtn}} onClick={() => setSelected({...ADMIN})}>{ADMIN.name}</button>
          </div>
        </>) : (
          <div style={s.pinArea}>
            <div style={s.pinWho}>
              <button style={s.backBtn} onClick={() => { setSelected(null); setPin(""); setError(false); }}>← 뒤로</button>
              <span style={s.pinName}>{selected.name}</span>
            </div>
            <p style={s.pinHint}>비밀번호 4자리를 입력하세요</p>
            <div style={s.pinDots}>
              {[0,1,2,3].map(i => (
                <div key={i} style={{...s.pinDot, ...(i < pin.length ? s.pinDotFilled : {}), ...(error ? s.pinDotError : {})}} />
              ))}
            </div>
            {error && <p style={s.pinError}>비밀번호가 틀렸습니다</p>}
            <div style={s.numPad}>
              {[1,2,3,4,5,6,7,8,9,"",0,"⌫"].map((k, i) => (
                <button key={i}
                  style={{...s.numKey, ...(k==="" ? s.numKeyEmpty : {})}}
                  onClick={() => {
                    if (k === "⌫") { setPin(p => p.slice(0,-1)); setError(false); }
                    else if (k !== "") handlePinInput(String(k));
                  }}
                  disabled={k === ""}
                >{k}</button>
              ))}
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

// ============================================================
// 🔝  Header
// ============================================================
function Header({ user, onLogout, screen, setScreen }) {
  const home = user?.role === "admin" ? "admin" : "home";
  return (
    <div style={s.header}>
      <div style={{display:"flex",alignItems:"center",gap:10}}>
        {screen !== home
          ? <button style={s.headerBackBtn} onClick={() => setScreen(home)}>← 뒤로</button>
          : <span style={s.headerTitle}>🧹 클린한사람들</span>
        }
      </div>
      <div style={s.headerRight}>
        <span style={s.headerUser}>{user?.name}</span>
        <button style={s.logoutBtn} onClick={onLogout}>로그아웃</button>
      </div>
    </div>
  );
}

// ============================================================
// 🧭  BottomNav
// ============================================================
function BottomNav({ role, screen, setScreen }) {
  const leaderNav = [
    {id:"home",      icon:"🏠", label:"홈"},
    {id:"submit",    icon:"✏️", label:"입력"},
    {id:"myreports", icon:"📋", label:"내역"},
    {id:"search",    icon:"🔍", label:"검색"},
  ];
  const adminNav = [
    {id:"admin",   icon:"📋", label:"전체보기"},
    {id:"stats",   icon:"📊", label:"통계"},
    {id:"search",  icon:"🔍", label:"검색"},
    {id:"members", icon:"👥", label:"팀장관리"},
  ];
  const items = role === "admin" ? adminNav : leaderNav;
  return (
    <div style={s.bottomNav}>
      {items.map(item => (
        <button key={item.id}
          style={{...s.navBtn, ...(screen===item.id ? s.navBtnOn : {})}}
          onClick={() => setScreen(item.id)}
        >
          <span style={{fontSize:22}}>{item.icon}</span>
          <span style={s.navLabel}>{item.label}</span>
        </button>
      ))}
    </div>
  );
}

// ============================================================
// 🏠  Home
// ============================================================
function HomeScreen({ user, reports, setScreen, onDelete, onEdit }) {
  const mine      = reports.filter(r => r.leader === user.name);
  const todayMine = mine.filter(r => r.date === todayStr());
  const thisMonth = monthOf(todayStr());
  const monthMine = mine.filter(r => monthOf(r.date) === thisMonth);
  const monthTotal = monthMine.reduce((a,r) => a + r.total, 0);

  return (
    <div style={s.page}>
      <div style={s.welcomeCard}>
        <span style={{fontSize:36}}>👋</span>
        <div>
          <p style={s.wName}>{user.name}</p>
          <p style={s.wSub}>오늘도 수고하십니다!</p>
        </div>
      </div>
      <div style={s.row2}>
        <div style={s.miniStat}><span style={s.miniStatLabel}>이번 달 합계</span><span style={s.miniStatValue}>{fmtKRW(monthTotal)}</span></div>
        <div style={s.miniStat}><span style={s.miniStatLabel}>이번 달 현장 수</span><span style={s.miniStatValue}>{monthMine.length}건</span></div>
      </div>
      <div style={s.menuGrid}>
        <button style={s.menuCard} onClick={() => setScreen("submit")}><span style={s.menuIcon}>✏️</span><span style={s.menuLbl}>현장 입력</span></button>
        <button style={s.menuCard} onClick={() => setScreen("myreports")}><span style={s.menuIcon}>📋</span><span style={s.menuLbl}>내 내역</span></button>
        <button style={s.menuCard} onClick={() => setScreen("search")}><span style={s.menuIcon}>🔍</span><span style={s.menuLbl}>현장 검색</span></button>
      </div>
      <p style={s.sectionTitle}>오늘 입력한 현장</p>
      {todayMine.length === 0
        ? <p style={s.empty}>오늘 입력된 현장이 없습니다.</p>
        : todayMine.map(r => <ReportCard key={r.id} r={r} canEdit onDelete={onDelete} onEdit={onEdit} />)
      }
    </div>
  );
}

// ============================================================
// ✏️  Submit
// ============================================================
function SubmitScreen({ user, reports, saveReports }) {
  const [form, setForm] = useState({ date:todayStr(), siteName:"", siteAddr:"", balance:"", extra:"", members:"1", memo:"" });
  const [photos, setPhotos]       = useState([]);
  const [done, setDone]           = useState(false);
  const [sending, setSending]     = useState(false);
  const [sendResult, setSendResult] = useState(null);
  const fileRef = useRef();
  const f = (k,v) => setForm(p => ({...p,[k]:v}));
  const total = Number(form.balance||0) + Number(form.extra||0);

  const handlePhotos = (e) => {
    const files = Array.from(e.target.files);
    const toAdd = files.slice(0, 50 - photos.length);
    toAdd.forEach(file => {
      const reader = new FileReader();
      reader.onload = ev => setPhotos(prev => [...prev, {name:file.name, dataUrl:ev.target.result}]);
      reader.readAsDataURL(file);
    });
    e.target.value = "";
  };

  const removePhoto = (idx) => setPhotos(prev => prev.filter((_,i) => i !== idx));

  const sendPhotosEmail = async () => {
    if (photos.length === 0) return alert("사진을 먼저 추가해주세요.");
    if (!form.siteName)       return alert("현장명을 먼저 입력해주세요.");
    if (GAS_URL.includes("여기에")) return alert("구글 스크립트 URL을 설정해주세요.");

    setSending(true); setSendResult(null);
    try {
      const photosData = photos.map((p,i) => ({
        filename: `${String(i+1).padStart(2,"0")}_${p.name}`,
        base64:   p.dataUrl.split(",")[1],
        mimeType: p.dataUrl.split(";")[0].replace("data:",""),
      }));
      const payload = JSON.stringify({
        to: ADMIN_EMAIL,
        subject: `[클린한사람들] ${user.name} - ${form.siteName} 현장사진 (${form.date})`,
        leader: user.name, siteName: form.siteName, siteAddr: form.siteAddr||"",
        date: form.date, balance: form.balance, extra: form.extra||"0",
        total: String(total), members: form.members, memo: form.memo||"",
        photos: photosData,
      });
      await fetch(GAS_URL, { method:"POST", mode:"no-cors", headers:{"Content-Type":"text/plain"}, body:payload });
      setSendResult("ok");
    } catch(e) {
      setSendResult("err");
    }
    setSending(false);
    setTimeout(() => setSendResult(null), 5000);
  };

  const submit = () => {
    if (!form.siteName || !form.balance) return alert("현장명과 잔금을 입력해주세요.");
    const r = {
      id: Date.now(), leader: user.name, date: form.date,
      siteName: form.siteName, siteAddr: form.siteAddr,
      balance: Number(form.balance), extra: Number(form.extra||0), total,
      members: Number(form.members||1), memo: form.memo,
      photoCount: photos.length, createdAt: new Date().toISOString(),
    };
    saveReports([r, ...reports]);
    setForm({ date:todayStr(), siteName:"", siteAddr:"", balance:"", extra:"", members:"1", memo:"" });
    setPhotos([]);
    setDone(true);
    setTimeout(() => setDone(false), 3000);
  };

  return (
    <div style={{position:"relative"}}>
      {done && (
        <div style={s.doneOverlay}>
          <div style={s.doneBox}>
            <div style={s.doneCircle}>✓</div>
            <p style={s.doneTitleText}>보고 되었습니다!</p>
            <p style={s.doneSubText}>현장 보고가 성공적으로 등록되었습니다.</p>
          </div>
        </div>
      )}
      <div style={s.page}>
        <h2 style={s.pageTitle}>현장 보고 입력</h2>
        <div style={s.card}>
          <Label>날짜</Label>
          <input type="date" style={s.input} value={form.date} onChange={e=>f("date",e.target.value)} />
          <Label>현장명 *</Label>
          <input type="text" style={s.input} placeholder="예) 강남구 OO아파트" value={form.siteName} onChange={e=>f("siteName",e.target.value)} />
          <Label>현장 주소</Label>
          <input type="text" style={s.input} placeholder="예) 서울시 강남구 역삼동 123-45" value={form.siteAddr} onChange={e=>f("siteAddr",e.target.value)} />
          <Label>잔금 (원) *</Label>
          <input type="number" style={s.input} placeholder="0" value={form.balance} onChange={e=>f("balance",e.target.value)} />
          <Label>추가금액 (원)</Label>
          <input type="number" style={s.input} placeholder="0" value={form.extra} onChange={e=>f("extra",e.target.value)} />
          <div style={s.totalBox}>
            <span style={{color:"#555"}}>합계</span>
            <span style={s.totalAmt}>{fmtKRW(total)}</span>
          </div>
          <Label>팀원 수 (본인 포함)</Label>
          <input type="number" style={s.input} min="1" value={form.members} onChange={e=>f("members",e.target.value)} />
          <Label>메모 (선택)</Label>
          <textarea style={{...s.input,height:80,resize:"none"}} placeholder="특이사항 등" value={form.memo} onChange={e=>f("memo",e.target.value)} />

          <div style={s.photoSection}>
            <div style={s.photoHeader}>
              <span style={s.photoTitle}>📸 현장 사진</span>
              <span style={s.photoCount}>{photos.length} / 50장</span>
            </div>
            {photos.length < 50 && <>
              <input ref={fileRef} type="file" accept="image/*" multiple style={{display:"none"}} onChange={handlePhotos} />
              <button style={s.photoAddBtn} onClick={() => fileRef.current.click()}>+ 사진 추가 (최대 50장)</button>
            </>}
            {photos.length > 0 && <>
              <div style={s.photoGrid}>
                {photos.map((p,i) => (
                  <div key={i} style={s.photoThumb}>
                    <img src={p.dataUrl} alt={p.name} style={s.thumbImg} />
                    <button style={s.thumbDel} onClick={() => removePhoto(i)}>✕</button>
                  </div>
                ))}
              </div>
              <button style={{...s.emailBtn, opacity:sending?0.7:1}} onClick={sendPhotosEmail} disabled={sending}>
                {sending ? "⏳ 전송 중..." : `📧 사진 이메일 전송 (${photos.length}장)`}
              </button>
              {sendResult === "ok"  && <div style={s.sendOk}>✅ 전송 완료! 잠시 후 관리자 메일함을 확인하세요.</div>}
              {sendResult === "err" && <div style={s.sendErr}>❌ 전송 실패. GAS_URL과 이메일 주소를 확인해주세요.</div>}
            </>}
          </div>

          <button style={s.submitBtn} onClick={submit}>보고 등록하기</button>
        </div>
      </div>
    </div>
  );
}

// ============================================================
// 📋  MyReports
// ============================================================
function MyReportsScreen({ user, reports, onDelete, onEdit }) {
  const [month, setMonth] = useState(todayStr().slice(0,7));
  const mine  = reports.filter(r => r.leader === user.name && monthOf(r.date) === month);
  const total = mine.reduce((a,r) => a+r.total, 0);
  return (
    <div style={s.page}>
      <h2 style={s.pageTitle}>내 보고 내역</h2>
      <div style={s.monthRow}>
        <input type="month" style={{...s.input,marginBottom:0,flex:1}} value={month} onChange={e=>setMonth(e.target.value)} />
      </div>
      {mine.length > 0 && (
        <div style={s.monthSummary}>
          <span>{mine.length}건 · 팀원 {mine.reduce((a,r)=>a+r.members,0)}명</span>
          <span style={{fontWeight:800,color:GREEN}}>{fmtKRW(total)}</span>
        </div>
      )}
      {mine.length === 0
        ? <p style={s.empty}>해당 월 데이터가 없습니다.</p>
        : mine.map(r => <ReportCard key={r.id} r={r} canEdit onDelete={onDelete} onEdit={onEdit} />)
      }
    </div>
  );
}

// ============================================================
// 🗂️  Admin
// ============================================================
function AdminScreen({ reports, leaders, loadAll, onDelete, onEdit }) {
  const [date,   setDate]   = useState("");
  const [leader, setLeader] = useState("");
  const leaderNames = leaders.map(l => l.name);
  const filtered = reports.filter(r => (!date || r.date===date) && (!leader || r.leader===leader));
  const sum = filtered.reduce((a,r) => ({
    balance: a.balance+r.balance, extra: a.extra+r.extra,
    total: a.total+r.total, members: a.members+r.members
  }), {balance:0,extra:0,total:0,members:0});

  const exportCSV = () => {
    const rows = [["날짜","팀장","현장명","현장주소","잔금","추가금액","합계","팀원수","메모"]];
    filtered.forEach(r => rows.push([r.date,r.leader,r.siteName,r.siteAddr||"",r.balance,r.extra,r.total,r.members,r.memo||""]));
    const csv = rows.map(r => r.map(c => `"${String(c).replace(/"/g,'""')}"`).join(",")).join("\n");
    const blob = new Blob(["\uFEFF"+csv], {type:"text/csv;charset=utf-8;"});
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = `현장보고_${date||"전체"}_${leader||"전팀장"}.csv`;
    a.click();
  };

  return (
    <div style={s.page}>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:14}}>
        <h2 style={{...s.pageTitle,marginBottom:0}}>전체 보고 현황</h2>
        <button style={s.refreshBtn} onClick={loadAll}>🔄 새로고침</button>
      </div>
      <div style={s.filterRow}>
        <input type="date" style={{...s.input,flex:1,marginBottom:0}} value={date} onChange={e=>setDate(e.target.value)} />
        <select style={{...s.input,flex:1,marginBottom:0}} value={leader} onChange={e=>setLeader(e.target.value)}>
          <option value="">전체 팀장</option>
          {leaderNames.map(n => <option key={n} value={n}>{n}</option>)}
        </select>
        <button style={s.clearBtn} onClick={() => { setDate(""); setLeader(""); }}>초기화</button>
      </div>
      <div style={s.summaryGrid}>
        <div style={s.sumCard}><span style={s.sumLbl}>잔금</span><span style={s.sumVal}>{fmtKRW(sum.balance)}</span></div>
        <div style={s.sumCard}><span style={s.sumLbl}>추가금</span><span style={s.sumVal}>{fmtKRW(sum.extra)}</span></div>
        <div style={{...s.sumCard,...s.sumCardTotal,gridColumn:"span 2"}}>
          <span style={{...s.sumLbl,color:"rgba(255,255,255,0.7)"}}>총 합계</span>
          <span style={{...s.sumVal,color:"#fff",fontSize:22}}>{fmtKRW(sum.total)}</span>
        </div>
        <div style={s.sumCard}><span style={s.sumLbl}>건수</span><span style={s.sumVal}>{filtered.length}건</span></div>
        <div style={s.sumCard}><span style={s.sumLbl}>총 인원</span><span style={s.sumVal}>{sum.members}명</span></div>
      </div>
      <button style={s.excelBtn} onClick={exportCSV}>📥 엑셀(CSV) 내보내기</button>
      {filtered.length === 0
        ? <p style={s.empty}>조회된 데이터가 없습니다.</p>
        : filtered.map(r => <ReportCard key={r.id} r={r} showLeader canEdit onDelete={onDelete} onEdit={onEdit} />)
      }
    </div>
  );
}

// ============================================================
// 📊  Stats
// ============================================================
function StatsScreen({ reports, user, leaders }) {
  const isAdmin = user.role === "admin";
  const [year,   setYear]   = useState(new Date().getFullYear());
  const [leader, setLeader] = useState(isAdmin ? "전체" : user.name);
  const leaderNames = leaders.map(l => l.name);
  const months = Array.from({length:12},(_,i)=>i+1);

  const filtered = reports.filter(r =>
    r.date?.startsWith(String(year)) && (leader==="전체" || r.leader===leader)
  );
  const byMonth = months.map(m => {
    const mStr = `${year}-${String(m).padStart(2,"0")}`;
    const mr = filtered.filter(r => r.date?.startsWith(mStr));
    return { m, label:`${m}월`, total:mr.reduce((a,r)=>a+r.total,0), count:mr.length, members:mr.reduce((a,r)=>a+r.members,0) };
  });
  const maxTotal  = Math.max(...byMonth.map(b=>b.total), 1);
  const yearTotal = byMonth.reduce((a,b)=>a+b.total, 0);

  return (
    <div style={s.page}>
      <h2 style={s.pageTitle}>월별 통계</h2>
      <div style={s.statsFilter}>
        <select style={{...s.input,marginBottom:0,flex:1}} value={year} onChange={e=>setYear(Number(e.target.value))}>
          {[2024,2025,2026,2027].map(y=><option key={y} value={y}>{y}년</option>)}
        </select>
        {isAdmin && (
          <select style={{...s.input,marginBottom:0,flex:1}} value={leader} onChange={e=>setLeader(e.target.value)}>
            <option value="전체">전체 팀장</option>
            {leaderNames.map(n=><option key={n} value={n}>{n}</option>)}
          </select>
        )}
      </div>
      <div style={s.yearTotal}>
        <span style={{opacity:0.8,fontSize:14}}>{year}년 연간 합계</span>
        <span style={{fontSize:22,fontWeight:900}}>{fmtKRW(yearTotal)}</span>
      </div>
      <div style={s.barChart}>
        {byMonth.map(b => (
          <div key={b.m} style={s.barCol}>
            <span style={s.barTopVal}>{b.total>0 ? fmtKRW(b.total).replace("원","") : ""}</span>
            <div style={s.barTrack}>
              <div style={{...s.barFill, height:b.total>0?`${Math.max(4,(b.total/maxTotal)*100)}%`:"4px", opacity:b.total>0?1:0.2}} />
            </div>
            <span style={s.barLabel}>{b.label}</span>
            <span style={s.barCount}>{b.count>0?`${b.count}건`:""}</span>
          </div>
        ))}
      </div>
      <div style={{marginTop:16}}>
        {byMonth.filter(b=>b.count>0).map(b => (
          <div key={b.m} style={s.statRow}>
            <div style={s.statMonth}>{b.label}</div>
            <div style={s.statDetails}>
              <span>{b.count}건</span>
              <span>팀원 {b.members}명</span>
              <span style={{fontWeight:800,color:GREEN}}>{fmtKRW(b.total)}</span>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}

// ============================================================
// 🔍  Search
// ============================================================
function SearchScreen({ reports, user, onDelete, onEdit }) {
  const [query, setQuery] = useState("");
  const isAdmin = user.role === "admin";

  const CHOSUNG = ["ㄱ","ㄲ","ㄴ","ㄷ","ㄸ","ㄹ","ㅁ","ㅂ","ㅃ","ㅅ","ㅆ","ㅇ","ㅈ","ㅉ","ㅊ","ㅋ","ㅌ","ㅍ","ㅎ"];
  const getChosung = str => str.split("").map(c => {
    const code = c.charCodeAt(0) - 0xAC00;
    return (code>=0 && code<=11171) ? CHOSUNG[Math.floor(code/588)] : c;
  }).join("");

  const fuzzyMatch = (text, q) => {
    if (!q.trim() || !text) return false;
    const t = text.toLowerCase(), qLow = q.toLowerCase().trim();
    if (t.includes(qLow)) return true;
    if (getChosung(text).includes(getChosung(q.trim()))) return true;
    return qLow.split(/\s+/).every(w => t.includes(w));
  };

  const pool = isAdmin ? reports : reports.filter(r => r.leader === user.name);
  const results = query.trim()
    ? pool.filter(r =>
        fuzzyMatch(r.siteName||"",query) || fuzzyMatch(r.siteAddr||"",query) ||
        fuzzyMatch(r.memo||"",query)     || fuzzyMatch(r.leader||"",query)
      ).sort((a,b) => b.date.localeCompare(a.date))
    : [];

  return (
    <div style={s.page}>
      <h2 style={s.pageTitle}>현장 검색</h2>
      <div style={s.searchBox}>
        <span style={{fontSize:18,marginRight:8}}>🔍</span>
        <input style={s.searchInput} placeholder="현장명, 주소, 메모로 검색..."
          value={query} onChange={e=>setQuery(e.target.value)} autoFocus />
        {query && <button style={s.searchClear} onClick={()=>setQuery("")}>✕</button>}
      </div>
      {!query.trim() && (
        <div style={s.searchHint}>
          <p style={s.searchHintTitle}>💡 이렇게 검색해보세요</p>
          <p style={s.searchHintText}>· 현장명 일부분만 입력해도 됩니다</p>
          <p style={s.searchHintText}>· 주소의 동/구 이름으로 검색 가능</p>
          <p style={s.searchHintText}>· 초성으로 검색 가능 (예: ㄱㄴ → 강남)</p>
          <p style={s.searchHintText}>· 메모 내용으로도 검색됩니다</p>
        </div>
      )}
      {query.trim() && results.length === 0 && (
        <div style={{textAlign:"center",padding:"50px 0"}}>
          <div style={{fontSize:40}}>🔎</div>
          <p style={{color:"#aaa",fontWeight:600}}>검색 결과가 없습니다</p>
        </div>
      )}
      {results.length > 0 && <>
        <p style={{fontSize:13,color:"#888",fontWeight:700,marginBottom:12}}>검색 결과 {results.length}건</p>
        {results.map(r => <ReportCard key={r.id} r={r} canEdit showLeader={isAdmin} onDelete={onDelete} onEdit={onEdit} highlight={query} />)}
      </>}
    </div>
  );
}

// ============================================================
// 👥  Members (관리자 전용)
// ============================================================
function MembersScreen({ leaders, saveLeaders }) {
  const [showAdd,    setShowAdd]    = useState(false);
  const [editTarget, setEditTarget] = useState(null);
  const [delTarget,  setDelTarget]  = useState(null);
  const [newName, setNewName] = useState("");
  const [newPin,  setNewPin]  = useState("");
  const [newPin2, setNewPin2] = useState("");
  const [addErr,  setAddErr]  = useState("");
  const [chgPin,  setChgPin]  = useState("");
  const [chgPin2, setChgPin2] = useState("");
  const [chgErr,  setChgErr]  = useState("");

  const handleAdd = () => {
    if (!newName.trim()) return setAddErr("이름을 입력해주세요.");
    if (leaders.find(l=>l.name===newName.trim())) return setAddErr("이미 존재하는 이름입니다.");
    if (!/^\d{4}$/.test(newPin))  return setAddErr("비밀번호는 숫자 4자리여야 합니다.");
    if (newPin !== newPin2)       return setAddErr("비밀번호가 일치하지 않습니다.");
    saveLeaders([...leaders, {name:newName.trim(), pin:newPin}]);
    setNewName(""); setNewPin(""); setNewPin2(""); setAddErr(""); setShowAdd(false);
  };

  const handleChangePIN = () => {
    if (!/^\d{4}$/.test(chgPin)) return setChgErr("비밀번호는 숫자 4자리여야 합니다.");
    if (chgPin !== chgPin2)      return setChgErr("비밀번호가 일치하지 않습니다.");
    saveLeaders(leaders.map(l => l.name===editTarget.name ? {...l,pin:chgPin} : l));
    setEditTarget(null); setChgPin(""); setChgPin2(""); setChgErr("");
  };

  return (
    <div style={s.page}>
      <h2 style={s.pageTitle}>팀장 관리</h2>
      <div style={s.memberList}>
        {leaders.length === 0 && <p style={s.empty}>등록된 팀장이 없습니다.</p>}
        {leaders.map((l,i) => (
          <div key={l.name} style={s.memberRow}>
            <div style={s.memberInfo}>
              <span style={s.memberNum}>{i+1}</span>
              <span style={s.memberName}>{l.name}</span>
              <span style={s.memberPin}>🔒 ●●●●</span>
            </div>
            <div style={s.memberActions}>
              <button style={s.pinChangeBtn} onClick={()=>{setEditTarget(l);setChgPin("");setChgPin2("");setChgErr("");}}>비번변경</button>
              <button style={s.memberDelBtn} onClick={()=>setDelTarget(l)}>삭제</button>
            </div>
          </div>
        ))}
      </div>

      {!showAdd
        ? <button style={s.addLeaderBtn} onClick={()=>{setShowAdd(true);setAddErr("");}}>+ 팀장 추가</button>
        : <div style={s.addBox}>
            <p style={s.addBoxTitle}>새 팀장 추가</p>
            <Label>이름</Label>
            <input style={s.input} placeholder="예) 홍팀장" value={newName} onChange={e=>{setNewName(e.target.value);setAddErr("");}} />
            <Label>비밀번호 (숫자 4자리)</Label>
            <input style={s.input} type="password" inputMode="numeric" maxLength={4} placeholder="●●●●" value={newPin} onChange={e=>{setNewPin(e.target.value);setAddErr("");}} />
            <Label>비밀번호 확인</Label>
            <input style={s.input} type="password" inputMode="numeric" maxLength={4} placeholder="●●●●" value={newPin2} onChange={e=>{setNewPin2(e.target.value);setAddErr("");}} />
            {addErr && <p style={s.formErr}>{addErr}</p>}
            <div style={{display:"flex",gap:10,marginTop:14}}>
              <button style={s.modalCancelBtn} onClick={()=>{setShowAdd(false);setNewName("");setNewPin("");setNewPin2("");setAddErr("");}}>취소</button>
              <button style={s.modalSaveBtn}   onClick={handleAdd}>추가하기</button>
            </div>
          </div>
      }

      {editTarget && (
        <div style={s.modalBg} onClick={()=>setEditTarget(null)}>
          <div style={{...s.modalBox}} onClick={e=>e.stopPropagation()}>
            <div style={s.modalHeader}>
              <span style={s.modalTitle}>{editTarget.name} 비밀번호 변경</span>
              <button style={s.modalClose} onClick={()=>setEditTarget(null)}>✕</button>
            </div>
            <div style={s.modalBody}>
              <Label>새 비밀번호 (숫자 4자리)</Label>
              <input style={s.input} type="password" inputMode="numeric" maxLength={4} placeholder="●●●●" value={chgPin} onChange={e=>{setChgPin(e.target.value);setChgErr("");}} />
              <Label>비밀번호 확인</Label>
              <input style={s.input} type="password" inputMode="numeric" maxLength={4} placeholder="●●●●" value={chgPin2} onChange={e=>{setChgPin2(e.target.value);setChgErr("");}} />
              {chgErr && <p style={s.formErr}>{chgErr}</p>}
            </div>
            <div style={s.modalFooter}>
              <button style={s.modalCancelBtn} onClick={()=>setEditTarget(null)}>취소</button>
              <button style={s.modalSaveBtn}   onClick={handleChangePIN}>변경하기</button>
            </div>
          </div>
        </div>
      )}

      {delTarget && (
        <div style={s.modalBg} onClick={()=>setDelTarget(null)}>
          <div style={{...s.modalBox,maxWidth:320}} onClick={e=>e.stopPropagation()}>
            <div style={{textAlign:"center",paddingTop:28}}><span style={{fontSize:48}}>⚠️</span></div>
            <p style={s.delTitle}>{delTarget.name}을 삭제할까요?</p>
            <p style={s.delSub}>삭제 후에도 해당 팀장의<br/>보고 데이터는 유지됩니다.</p>
            <div style={s.modalFooter}>
              <button style={s.modalCancelBtn} onClick={()=>setDelTarget(null)}>취소</button>
              <button style={s.delConfirmBtn} onClick={()=>{saveLeaders(leaders.filter(l=>l.name!==delTarget.name));setDelTarget(null);}}>삭제</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

// ============================================================
// ✏️  EditModal
// ============================================================
function EditModal({ r, onSave, onClose }) {
  const [form, setForm] = useState({
    date:r.date, siteName:r.siteName, siteAddr:r.siteAddr||"",
    balance:String(r.balance), extra:String(r.extra),
    members:String(r.members), memo:r.memo||"",
  });
  const f = (k,v) => setForm(p=>({...p,[k]:v}));
  const total = Number(form.balance||0) + Number(form.extra||0);

  const handleSave = () => {
    if (!form.siteName || !form.balance) return alert("현장명과 잔금을 입력해주세요.");
    onSave({...r, date:form.date, siteName:form.siteName, siteAddr:form.siteAddr,
      balance:Number(form.balance), extra:Number(form.extra||0), total,
      members:Number(form.members||1), memo:form.memo});
  };

  return (
    <div style={s.modalBg} onClick={onClose}>
      <div style={s.modalBox} onClick={e=>e.stopPropagation()}>
        <div style={s.modalHeader}>
          <span style={s.modalTitle}>현장 수정</span>
          <button style={s.modalClose} onClick={onClose}>✕</button>
        </div>
        <div style={s.modalBody}>
          <Label>날짜</Label><input type="date" style={s.input} value={form.date} onChange={e=>f("date",e.target.value)} />
          <Label>현장명 *</Label><input type="text" style={s.input} value={form.siteName} onChange={e=>f("siteName",e.target.value)} />
          <Label>현장 주소</Label><input type="text" style={s.input} value={form.siteAddr} onChange={e=>f("siteAddr",e.target.value)} />
          <Label>잔금 (원) *</Label><input type="number" style={s.input} value={form.balance} onChange={e=>f("balance",e.target.value)} />
          <Label>추가금액 (원)</Label><input type="number" style={s.input} value={form.extra} onChange={e=>f("extra",e.target.value)} />
          <div style={s.totalBox}><span style={{color:"#555"}}>합계</span><span style={s.totalAmt}>{fmtKRW(total)}</span></div>
          <Label>팀원 수</Label><input type="number" style={s.input} min="1" value={form.members} onChange={e=>f("members",e.target.value)} />
          <Label>메모</Label><textarea style={{...s.input,height:70,resize:"none"}} value={form.memo} onChange={e=>f("memo",e.target.value)} />
        </div>
        <div style={s.modalFooter}>
          <button style={s.modalCancelBtn} onClick={onClose}>취소</button>
          <button style={s.modalSaveBtn}   onClick={handleSave}>저장하기</button>
        </div>
      </div>
    </div>
  );
}

// ============================================================
// 🗑️  DeleteConfirm
// ============================================================
function DeleteConfirm({ r, onConfirm, onClose }) {
  return (
    <div style={s.modalBg} onClick={onClose}>
      <div style={{...s.modalBox,maxWidth:320}} onClick={e=>e.stopPropagation()}>
        <div style={{textAlign:"center",paddingTop:28}}><span style={{fontSize:48}}>🗑️</span></div>
        <p style={s.delTitle}>보고를 삭제할까요?</p>
        <p style={s.delSub}><strong>{r.siteName}</strong> ({r.date})<br/>이 작업은 되돌릴 수 없습니다.</p>
        <div style={s.modalFooter}>
          <button style={s.modalCancelBtn} onClick={onClose}>취소</button>
          <button style={s.delConfirmBtn}  onClick={()=>{onConfirm(r.id);onClose();}}>삭제</button>
        </div>
      </div>
    </div>
  );
}

// ============================================================
// 🃏  ReportCard
// ============================================================
function Highlight({ text, query }) {
  if (!query || !text) return <span>{text}</span>;
  const idx = text.toLowerCase().indexOf(query.toLowerCase());
  if (idx === -1) return <span>{text}</span>;
  return <span>{text.slice(0,idx)}<mark style={{background:"#fffb8f",borderRadius:2,padding:"0 1px"}}>{text.slice(idx,idx+query.length)}</mark>{text.slice(idx+query.length)}</span>;
}

function ReportCard({ r, showLeader, canEdit, onDelete, onEdit, highlight }) {
  const [showEdit, setShowEdit] = useState(false);
  const [showDel,  setShowDel]  = useState(false);
  return (
    <>
      {showEdit && <EditModal r={r} onSave={u=>{onEdit(u);setShowEdit(false);}} onClose={()=>setShowEdit(false)} />}
      {showDel  && <DeleteConfirm r={r} onConfirm={onDelete} onClose={()=>setShowDel(false)} />}
      <div style={s.rcard}>
        <div style={s.rcardHead}>
          <div style={{flex:1,minWidth:0}}>
            <span style={s.rcardSite}><Highlight text={r.siteName} query={highlight} /></span>
            {showLeader && <span style={s.rcardTag}>{r.leader}</span>}
          </div>
          <div style={{display:"flex",alignItems:"center",gap:6,flexShrink:0}}>
            <span style={s.rcardDate}>{r.date}</span>
            {canEdit && <>
              <button style={s.editBtn} onClick={()=>setShowEdit(true)}>✏️</button>
              <button style={s.delBtn}  onClick={()=>setShowDel(true)}>🗑️</button>
            </>}
          </div>
        </div>
        {r.siteAddr && <div style={s.rcardAddr}>📍 <Highlight text={r.siteAddr} query={highlight} /></div>}
        <div style={s.rcardBody}>
          <Row label="잔금"    value={fmtKRW(r.balance)} />
          <Row label="추가금"  value={fmtKRW(r.extra)} />
          <div style={{...s.rrow,fontWeight:800,color:GREEN,borderTop:"1px dashed #d4edd9",paddingTop:8}}>
            <span>합계</span><span>{fmtKRW(r.total)}</span>
          </div>
          <Row label="👥 팀원수" value={`${r.members}명`} />
          {r.photoCount > 0 && <Row label="📸 사진" value={`${r.photoCount}장 첨부됨`} />}
          {r.memo && <div style={s.memoBox}>📝 <Highlight text={r.memo} query={highlight} /></div>}
        </div>
      </div>
    </>
  );
}

function Row({ label, value }) {
  return <div style={s.rrow}><span style={{color:"#888"}}>{label}</span><span>{value}</span></div>;
}
function Label({ children }) {
  return <p style={s.label}>{children}</p>;
}

// ============================================================
// 🎨  Styles
// ============================================================
const s = {
  app:           { display:"flex", flexDirection:"column", height:"100vh", background:"#f2f5f3", fontFamily:"'Noto Sans KR',sans-serif", maxWidth:480, margin:"0 auto", position:"relative", overflow:"hidden" },
  content:       { flex:1, overflowY:"auto", paddingBottom:80 },

  // Login
  loginBg:       { minHeight:"100vh", background:`linear-gradient(160deg,${GREEN} 0%,${LGREEN} 100%)`, display:"flex", alignItems:"center", justifyContent:"center", padding:20 },
  loginCard:     { background:"#fff", borderRadius:28, padding:"36px 24px", width:"100%", maxWidth:380, boxShadow:"0 24px 60px rgba(0,0,0,0.3)" },
  logoArea:      { textAlign:"center", marginBottom:28 },
  logoTitle:     { margin:"8px 0 4px", fontSize:30, fontWeight:900, color:GREEN },
  logoSub:       { margin:0, color:"#888", fontSize:14 },
  selectHint:    { textAlign:"center", color:"#555", fontWeight:600, marginBottom:12, marginTop:0 },
  groupLabel:    { fontSize:12, fontWeight:700, color:"#aaa", letterSpacing:1, marginBottom:8, marginTop:16 },
  nameGrid:      { display:"grid", gridTemplateColumns:"repeat(3,1fr)", gap:8 },
  nameBtn:       { padding:"12px 4px", borderRadius:12, border:"2px solid #d4eedd", background:"#f0faf4", color:GREEN, fontWeight:700, fontSize:14, cursor:"pointer" },
  adminNameBtn:  { background:"#fff8ef", border:"2px solid #ffe0b0", color:"#b45000" },

  // PIN
  pinArea:       { paddingTop:8 },
  pinWho:        { display:"flex", alignItems:"center", marginBottom:16 },
  backBtn:       { background:"none", border:"none", color:"#888", cursor:"pointer", fontSize:14, marginRight:12 },
  pinName:       { fontWeight:800, fontSize:20, color:GREEN },
  pinHint:       { textAlign:"center", color:"#888", fontSize:14, marginBottom:20 },
  pinDots:       { display:"flex", justifyContent:"center", gap:16, marginBottom:8 },
  pinDot:        { width:18, height:18, borderRadius:"50%", border:"2px solid #ccc", background:"#eee", transition:"all 0.15s" },
  pinDotFilled:  { background:GREEN, border:`2px solid ${GREEN}` },
  pinDotError:   { background:"#e53935", borderColor:"#e53935" },
  pinError:      { textAlign:"center", color:"#e53935", fontWeight:700, fontSize:13, margin:"0 0 12px" },
  numPad:        { display:"grid", gridTemplateColumns:"repeat(3,1fr)", gap:10, marginTop:20 },
  numKey:        { padding:"16px 0", borderRadius:12, border:"1px solid #e8e8e8", background:"#fafafa", fontSize:22, fontWeight:600, cursor:"pointer" },
  numKeyEmpty:   { background:"none", border:"none", cursor:"default" },

  // Header
  header:        { background:GREEN, color:"#fff", padding:"14px 20px", display:"flex", alignItems:"center", justifyContent:"space-between", flexShrink:0 },
  headerTitle:   { fontWeight:900, fontSize:18 },
  headerRight:   { display:"flex", alignItems:"center", gap:10 },
  headerUser:    { fontSize:13, opacity:0.75 },
  logoutBtn:     { background:"rgba(255,255,255,0.2)", border:"none", color:"#fff", padding:"6px 14px", borderRadius:20, fontSize:13, cursor:"pointer" },
  headerBackBtn: { background:"rgba(255,255,255,0.2)", border:"none", color:"#fff", padding:"7px 16px", borderRadius:20, fontSize:14, fontWeight:700, cursor:"pointer" },

  // Nav
  bottomNav:     { position:"fixed", bottom:0, left:"50%", transform:"translateX(-50%)", width:"100%", maxWidth:480, background:"#fff", borderTop:"1px solid #e5e5e5", display:"flex", padding:"8px 0", boxShadow:"0 -4px 20px rgba(0,0,0,0.08)" },
  navBtn:        { flex:1, display:"flex", flexDirection:"column", alignItems:"center", gap:2, background:"none", border:"none", cursor:"pointer", color:"#bbb", padding:"4px 0" },
  navBtnOn:      { color:GREEN },
  navLabel:      { fontSize:10, fontWeight:700 },

  // Page
  page:          { padding:"20px 16px" },
  pageTitle:     { fontSize:22, fontWeight:900, color:GREEN, marginBottom:16, marginTop:0 },
  sectionTitle:  { fontSize:15, fontWeight:800, color:"#444", margin:"20px 0 10px" },
  empty:         { color:"#bbb", textAlign:"center", padding:"30px 0" },

  // Home
  welcomeCard:   { background:`linear-gradient(135deg,${GREEN},${LGREEN})`, borderRadius:20, padding:"20px 24px", display:"flex", alignItems:"center", gap:16, marginBottom:16, color:"#fff" },
  wName:         { margin:0, fontWeight:900, fontSize:22 },
  wSub:          { margin:"4px 0 0", opacity:0.8, fontSize:14 },
  row2:          { display:"grid", gridTemplateColumns:"1fr 1fr", gap:10, marginBottom:16 },
  miniStat:      { background:"#fff", borderRadius:14, padding:"14px 16px", display:"flex", flexDirection:"column", gap:4, boxShadow:"0 2px 8px rgba(0,0,0,0.06)" },
  miniStatLabel: { fontSize:12, color:"#888", fontWeight:600 },
  miniStatValue: { fontSize:16, fontWeight:800, color:GREEN },
  menuGrid:      { display:"grid", gridTemplateColumns:"repeat(3,1fr)", gap:10, marginBottom:4 },
  menuCard:      { background:"#fff", border:"none", borderRadius:16, padding:"20px 8px", display:"flex", flexDirection:"column", alignItems:"center", gap:8, cursor:"pointer", boxShadow:"0 2px 10px rgba(0,0,0,0.07)" },
  menuIcon:      { fontSize:28 },
  menuLbl:       { fontSize:13, fontWeight:700, color:"#333" },

  // Form
  card:          { background:"#fff", borderRadius:18, padding:"20px 18px", boxShadow:"0 2px 12px rgba(0,0,0,0.08)" },
  label:         { fontSize:13, fontWeight:700, color:"#555", margin:"14px 0 6px 0" },
  input:         { width:"100%", padding:"12px 14px", border:"1.5px solid #e0e0e0", borderRadius:10, fontSize:15, outline:"none", boxSizing:"border-box", fontFamily:"inherit", background:"#fafafa", display:"block", marginBottom:2 },
  totalBox:      { background:"#eaf6ef", borderRadius:12, padding:"14px 18px", display:"flex", justifyContent:"space-between", alignItems:"center", marginTop:10, fontWeight:700 },
  totalAmt:      { fontSize:20, color:GREEN, fontWeight:900 },
  submitBtn:     { marginTop:20, width:"100%", padding:16, background:`linear-gradient(135deg,${GREEN},${LGREEN})`, color:"#fff", border:"none", borderRadius:12, fontSize:16, fontWeight:800, cursor:"pointer" },

  // Done
  doneOverlay:   { position:"fixed", top:0, left:0, right:0, bottom:0, background:"rgba(0,0,0,0.5)", zIndex:999, display:"flex", alignItems:"center", justifyContent:"center" },
  doneBox:       { background:"#fff", borderRadius:28, padding:"48px 36px", textAlign:"center", boxShadow:"0 24px 60px rgba(0,0,0,0.3)", maxWidth:300, width:"90%" },
  doneCircle:    { width:72, height:72, borderRadius:"50%", background:`linear-gradient(135deg,${GREEN},${LGREEN})`, color:"#fff", fontSize:36, display:"flex", alignItems:"center", justifyContent:"center", margin:"0 auto 20px" },
  doneTitleText: { fontSize:22, fontWeight:900, color:GREEN, margin:"0 0 8px" },
  doneSubText:   { fontSize:14, color:"#888", margin:0 },

  // Month
  monthRow:      { display:"flex", gap:8, marginBottom:12 },
  monthSummary:  { background:"#eaf6ef", borderRadius:10, padding:"10px 16px", display:"flex", justifyContent:"space-between", marginBottom:12, fontSize:14 },

  // Admin
  filterRow:     { display:"flex", gap:8, marginBottom:14, alignItems:"center" },
  clearBtn:      { padding:"12px 12px", background:"#eee", border:"none", borderRadius:10, fontWeight:700, cursor:"pointer", whiteSpace:"nowrap", fontSize:13 },
  refreshBtn:    { background:"#eaf6ef", border:"none", borderRadius:10, padding:"8px 14px", color:GREEN, fontWeight:700, cursor:"pointer", fontSize:13 },
  summaryGrid:   { display:"grid", gridTemplateColumns:"1fr 1fr", gap:10, marginBottom:14 },
  sumCard:       { background:"#fff", borderRadius:14, padding:"14px 16px", display:"flex", flexDirection:"column", gap:4, boxShadow:"0 2px 8px rgba(0,0,0,0.06)" },
  sumCardTotal:  { background:GREEN },
  sumLbl:        { fontSize:12, color:"#888", fontWeight:600 },
  sumVal:        { fontSize:17, fontWeight:800, color:GREEN },
  excelBtn:      { width:"100%", padding:"13px 0", background:"#e8f5e9", color:"#2e7d32", border:"2px solid #a5d6a7", borderRadius:12, fontWeight:800, fontSize:14, cursor:"pointer", marginBottom:16 },

  // Stats
  statsFilter:   { display:"flex", gap:8, marginBottom:14 },
  yearTotal:     { background:`linear-gradient(135deg,${GREEN},${LGREEN})`, color:"#fff", borderRadius:16, padding:"18px 24px", display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:20 },
  barChart:      { display:"flex", alignItems:"flex-end", gap:4, height:180, background:"#fff", borderRadius:16, padding:"16px 12px 12px", boxShadow:"0 2px 10px rgba(0,0,0,0.07)", marginBottom:16, overflowX:"auto" },
  barCol:        { flex:1, display:"flex", flexDirection:"column", alignItems:"center", gap:2, minWidth:30 },
  barTopVal:     { fontSize:8, color:"#888", fontWeight:600, textAlign:"center", whiteSpace:"nowrap" },
  barTrack:      { width:"100%", height:100, display:"flex", alignItems:"flex-end" },
  barFill:       { width:"100%", background:`linear-gradient(to top,${GREEN},${LGREEN})`, borderRadius:"4px 4px 0 0", transition:"height 0.4s" },
  barLabel:      { fontSize:11, color:"#555", fontWeight:700 },
  barCount:      { fontSize:9, color:"#aaa" },
  statRow:       { background:"#fff", borderRadius:12, padding:"12px 16px", marginBottom:8, display:"flex", justifyContent:"space-between", alignItems:"center", boxShadow:"0 1px 6px rgba(0,0,0,0.06)" },
  statMonth:     { fontWeight:800, color:GREEN, width:36 },
  statDetails:   { display:"flex", gap:16, fontSize:14 },

  // Search
  searchBox:     { display:"flex", alignItems:"center", background:"#fff", borderRadius:14, padding:"4px 16px", boxShadow:"0 2px 12px rgba(0,0,0,0.09)", marginBottom:16, border:"1.5px solid #e0e0e0" },
  searchInput:   { flex:1, border:"none", outline:"none", fontSize:16, padding:"12px 0", fontFamily:"inherit", background:"transparent" },
  searchClear:   { background:"none", border:"none", color:"#aaa", fontSize:18, cursor:"pointer", padding:"4px" },
  searchHint:    { background:"#f8fbf9", borderRadius:14, padding:"18px 20px" },
  searchHintTitle: { fontWeight:800, color:GREEN, margin:"0 0 10px", fontSize:15 },
  searchHintText:  { color:"#888", fontSize:14, margin:"4px 0" },

  // Photo
  photoSection:  { marginTop:20, borderTop:"1.5px dashed #e0e0e0", paddingTop:16 },
  photoHeader:   { display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:10 },
  photoTitle:    { fontWeight:800, fontSize:15, color:"#333" },
  photoCount:    { fontSize:13, color:"#888", fontWeight:600 },
  photoAddBtn:   { width:"100%", padding:"13px 0", background:"#f0faf4", border:"2px dashed #a5d6b8", borderRadius:12, color:GREEN, fontWeight:700, fontSize:14, cursor:"pointer", marginBottom:12 },
  photoGrid:     { display:"grid", gridTemplateColumns:"repeat(4,1fr)", gap:8, marginBottom:12 },
  photoThumb:    { position:"relative", aspectRatio:"1", borderRadius:10, overflow:"hidden" },
  thumbImg:      { width:"100%", height:"100%", objectFit:"cover" },
  thumbDel:      { position:"absolute", top:3, right:3, background:"rgba(0,0,0,0.55)", border:"none", color:"#fff", borderRadius:"50%", width:20, height:20, fontSize:11, cursor:"pointer", display:"flex", alignItems:"center", justifyContent:"center" },
  emailBtn:      { width:"100%", padding:"14px 0", background:"linear-gradient(135deg,#1a73e8,#0d47a1)", color:"#fff", border:"none", borderRadius:12, fontWeight:800, fontSize:14, cursor:"pointer", marginBottom:8 },
  sendOk:        { background:"#e8f5e9", color:"#2e7d32", borderRadius:10, padding:"10px 14px", fontSize:13, fontWeight:700, textAlign:"center", marginBottom:6 },
  sendErr:       { background:"#fdecea", color:"#c62828", borderRadius:10, padding:"10px 14px", fontSize:13, fontWeight:700, textAlign:"center", marginBottom:6 },

  // Members
  memberList:    { background:"#fff", borderRadius:16, overflow:"hidden", boxShadow:"0 2px 10px rgba(0,0,0,0.07)", marginBottom:16 },
  memberRow:     { display:"flex", alignItems:"center", justifyContent:"space-between", padding:"14px 16px", borderBottom:"1px solid #f0f0f0" },
  memberInfo:    { display:"flex", alignItems:"center", gap:10 },
  memberNum:     { width:24, height:24, borderRadius:"50%", background:"#eaf6ef", color:GREEN, fontSize:12, fontWeight:800, display:"flex", alignItems:"center", justifyContent:"center" },
  memberName:    { fontWeight:800, fontSize:16, color:"#222" },
  memberPin:     { fontSize:12, color:"#aaa", marginLeft:4 },
  memberActions: { display:"flex", gap:8 },
  pinChangeBtn:  { padding:"6px 12px", background:"#eaf6ef", border:"none", borderRadius:8, color:GREEN, fontWeight:700, fontSize:12, cursor:"pointer" },
  memberDelBtn:  { padding:"6px 12px", background:"#fdecea", border:"none", borderRadius:8, color:"#e53935", fontWeight:700, fontSize:12, cursor:"pointer" },
  addLeaderBtn:  { width:"100%", padding:"16px 0", background:`linear-gradient(135deg,${GREEN},${LGREEN})`, color:"#fff", border:"none", borderRadius:14, fontSize:16, fontWeight:800, cursor:"pointer" },
  addBox:        { background:"#fff", borderRadius:16, padding:"20px 18px", boxShadow:"0 2px 12px rgba(0,0,0,0.08)" },
  addBoxTitle:   { fontSize:17, fontWeight:900, color:GREEN, margin:"0 0 4px" },
  formErr:       { color:"#e53935", fontSize:13, fontWeight:700, margin:"6px 0 0" },

  // Report card
  editBtn:       { background:"#eaf6ef", border:"none", borderRadius:8, padding:"4px 8px", fontSize:14, cursor:"pointer" },
  delBtn:        { background:"#fdecea", border:"none", borderRadius:8, padding:"4px 8px", fontSize:14, cursor:"pointer" },
  rcard:         { background:"#fff", borderRadius:16, marginBottom:12, overflow:"hidden", boxShadow:"0 2px 10px rgba(0,0,0,0.07)" },
  rcardHead:     { background:"#f6fbf7", padding:"12px 16px", display:"flex", justifyContent:"space-between", alignItems:"flex-start", borderBottom:"1px solid #eee" },
  rcardSite:     { fontWeight:900, fontSize:16, color:GREEN },
  rcardTag:      { marginLeft:8, background:"#eaf6ef", color:LGREEN, padding:"2px 8px", borderRadius:20, fontSize:12, fontWeight:700 },
  rcardDate:     { fontSize:13, color:"#999" },
  rcardAddr:     { padding:"6px 16px", background:"#fafafa", fontSize:13, color:"#666", borderBottom:"1px solid #f0f0f0" },
  rcardBody:     { padding:"12px 16px" },
  rrow:          { display:"flex", justifyContent:"space-between", padding:"5px 0", fontSize:14 },
  memoBox:       { marginTop:10, background:"#fffbf0", border:"1px solid #ffe0b0", borderRadius:8, padding:"8px 12px", fontSize:13, color:"#7a5500" },

  // Modal
  modalBg:       { position:"fixed", top:0, left:0, right:0, bottom:0, background:"rgba(0,0,0,0.55)", zIndex:1000, display:"flex", alignItems:"flex-end", justifyContent:"center" },
  modalBox:      { background:"#fff", borderRadius:"24px 24px 0 0", width:"100%", maxWidth:480, maxHeight:"90vh", overflow:"hidden", display:"flex", flexDirection:"column" },
  modalHeader:   { display:"flex", justifyContent:"space-between", alignItems:"center", padding:"18px 20px 12px", borderBottom:"1px solid #eee" },
  modalTitle:    { fontSize:18, fontWeight:900, color:GREEN },
  modalClose:    { background:"#f0f0f0", border:"none", borderRadius:"50%", width:32, height:32, cursor:"pointer", fontSize:16 },
  modalBody:     { overflowY:"auto", padding:"4px 20px 12px", flex:1 },
  modalFooter:   { display:"flex", gap:10, padding:"14px 20px", borderTop:"1px solid #eee" },
  modalCancelBtn:{ flex:1, padding:14, background:"#f0f0f0", border:"none", borderRadius:12, fontWeight:700, fontSize:15, cursor:"pointer" },
  modalSaveBtn:  { flex:2, padding:14, background:`linear-gradient(135deg,${GREEN},${LGREEN})`, color:"#fff", border:"none", borderRadius:12, fontWeight:800, fontSize:15, cursor:"pointer" },
  delTitle:      { textAlign:"center", fontSize:18, fontWeight:900, color:"#333", margin:"10px 0 6px" },
  delSub:        { textAlign:"center", fontSize:14, color:"#888", margin:"0 0 8px", lineHeight:1.6 },
  delConfirmBtn: { flex:2, padding:14, background:"#e53935", color:"#fff", border:"none", borderRadius:12, fontWeight:800, fontSize:15, cursor:"pointer" },
};

// ============================================================
// 🚀  Mount
// ============================================================
ReactDOM.createRoot(document.getElementById("root")).render(<App />);
</script>
</body>
</html>
