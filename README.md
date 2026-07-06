import { useState, useEffect, useCallback } from "react";
import { initializeApp } from "firebase/app";
import {
  getAuth, signInWithEmailAndPassword, signOut, onAuthStateChanged,
  createUserWithEmailAndPassword, updatePassword
} from "firebase/auth";
import {
  getFirestore, collection, doc, getDoc, getDocs, setDoc, updateDoc,
  deleteDoc, onSnapshot, serverTimestamp
} from "firebase/firestore";

// ─── FIREBASE INIT ────────────────────────────────────────────────────────────
const firebaseConfig = {
  apiKey: "AIzaSyCvxxnpdlmCFHpG70VvMXMVs5BrwfxH760",
  authDomain: "falco-connect.firebaseapp.com",
  projectId: "falco-connect",
  storageBucket: "falco-connect.firebasestorage.app",
  messagingSenderId: "405280342775",
  appId: "1:405280342775:web:d05ee889620ef3ad8a89f2"
};
const fbApp = initializeApp(firebaseConfig);
const auth  = getAuth(fbApp);
const db    = getFirestore(fbApp);

// ─── CRM FIREBASE ─────────────────────────────────────────────────────────────
const crmConfig = {
  apiKey: "AIzaSyATSFpQZ3phjtt7JyBSCh2GbofQN-segyo",
  authDomain: "falco-crm-f3768.firebaseapp.com",
  projectId: "falco-crm-f3768",
  storageBucket: "falco-crm-f3768.firebasestorage.app",
  messagingSenderId: "967154142883",
  appId: "1:967154142883:web:f54dd6cf5b6cb8d8003916"
};
const crmApp = initializeApp(crmConfig, "crm");
const crmDb  = getFirestore(crmApp);

// ─── THEME ───────────────────────────────────────────────────────────────────
const GOLD       = "#E53935";
const GOLD_LIGHT = "#FF6F6F";
const GOLD_DARK  = "#B71C1C";
const BLACK      = "#0D0D0D";
const BLACK2     = "#141414";
const BLACK3     = "#1C1C1C";
const WHITE      = "#FFFFFF";
const GRAY       = "#888888";

const ROLES = [
  { value: "super_admin",     label: "Super Admin",     pages: ["dashboard","employees","departments","attendance","payroll","leaves","sales","leaderboard","settings"] },
  { value: "ceo",             label: "CEO",             pages: ["dashboard","employees","departments","attendance","payroll","leaves","sales","leaderboard","settings"] },
  { value: "manager",         label: "Manager",         pages: ["dashboard","departments","employees","attendance","leaves","payroll","crm"] },
  { value: "hr_manager",      label: "HR Manager",      pages: ["dashboard","employees","attendance","leaves","payroll"] },
  { value: "finance_manager", label: "Finance Manager", pages: ["dashboard","payroll","employees"] },
  { value: "employee",        label: "Employee",        pages: ["dashboard","attendance","leaves","payroll"] },
];

// Roles that can add employees and create login credentials
const CAN_ADD_EMPLOYEES = ["super_admin","ceo","manager","hr_manager"];

async function seedSuperAdmin() {
  try {
    const ref  = doc(db, "users", "admin@falcoconnect.com");
    const snap = await getDoc(ref);
    if (!snap.exists()) {
      await setDoc(ref, {
        name: "Falco Connect Admin", email: "admin@falcoconnect.com",
        role: "super_admin", department: "", active: true,
        createdAt: serverTimestamp(),
      });
    }
  } catch(e) { console.log("Seed skipped:", e.message); }
}

// ─── GLOBAL CSS ───────────────────────────────────────────────────────────────
const GlobalStyle = () => (
  <style>{`
    @import url('https://fonts.googleapis.com/css2?family=Cinzel:wght@400;600;700;900&family=Rajdhani:wght@300;400;500;600;700&display=swap');
    *, *::before, *::after { margin:0; padding:0; box-sizing:border-box; }
    html, body, #root { height:100%; }
    html, body, #root { background:#0D0D0D !important; color:${WHITE}; font-family:'Rajdhani',sans-serif; min-height:100vh; }
    ::-webkit-scrollbar{width:4px;}
    ::-webkit-scrollbar-track{background:${BLACK2};}
    ::-webkit-scrollbar-thumb{background:${GOLD_DARK};border-radius:2px;}
    @keyframes fadeIn{from{opacity:0;transform:translateY(10px)}to{opacity:1;transform:translateY(0)}}
    @keyframes spin{from{transform:rotate(0deg)}to{transform:rotate(360deg)}}
    .fade{animation:fadeIn .35s ease forwards;}
    .spin{animation:spin 1s linear infinite;}
    .gold-btn{background:linear-gradient(135deg,${GOLD},${GOLD_LIGHT},${GOLD});background-size:200% auto;border:none;color:${BLACK};padding:10px 24px;border-radius:4px;cursor:pointer;font-family:'Rajdhani',sans-serif;font-size:15px;font-weight:700;letter-spacing:1.5px;transition:all .3s;text-transform:uppercase;}
    .gold-btn:hover{background-position:right center;box-shadow:0 4px 20px rgba(201,168,76,.4);transform:translateY(-1px);}
    .gold-btn:disabled{opacity:.5;cursor:not-allowed;transform:none;}
    .ghost-btn{background:transparent;border:1px solid ${GOLD};color:${GOLD};padding:8px 18px;border-radius:4px;cursor:pointer;font-family:'Rajdhani',sans-serif;font-size:13px;font-weight:600;letter-spacing:1px;transition:all .2s;}
    .ghost-btn:hover{background:${GOLD};color:${BLACK};}
    .danger-btn{background:transparent;border:1px solid #C94C4C;color:#C94C4C;padding:6px 12px;border-radius:4px;cursor:pointer;font-family:'Rajdhani',sans-serif;font-size:12px;font-weight:600;transition:all .2s;}
    .danger-btn:hover{background:#C94C4C;color:#fff;}
    .drv-input{background:${BLACK3} !important;border:1px solid #444 !important;color:${WHITE} !important;padding:11px 14px !important;border-radius:4px;font-family:'Rajdhani',sans-serif;font-size:15px;width:100%;outline:none;transition:border-color .2s;-webkit-text-fill-color:${WHITE} !important;}
    .drv-input:focus{border-color:${GOLD} !important;}
    .drv-input::placeholder{color:${GRAY} !important;-webkit-text-fill-color:${GRAY} !important;}
    .drv-input:-webkit-autofill,.drv-input:-webkit-autofill:hover,.drv-input:-webkit-autofill:focus{-webkit-box-shadow:0 0 0 1000px ${BLACK3} inset !important;-webkit-text-fill-color:${WHITE} !important;border-color:#444 !important;}
    .drv-select{background:${BLACK3};border:1px solid #444;color:${WHITE};padding:11px 14px;border-radius:4px;font-family:'Rajdhani',sans-serif;font-size:15px;width:100%;outline:none;transition:border-color .2s;appearance:none;cursor:pointer;}
    .drv-select:focus{border-color:${GOLD};}
    .drv-select option{background:${BLACK3};color:${WHITE};}
    .card{background:${BLACK2};border:1px solid #222;border-radius:8px;padding:20px;}
    .hov{transition:all .2s;cursor:pointer;}
    .hov:hover{border-color:${GOLD_DARK};transform:translateY(-2px);box-shadow:0 8px 24px rgba(0,0,0,.4);}
    .badge{display:inline-block;padding:3px 10px;border-radius:20px;font-size:12px;font-weight:600;letter-spacing:.5px;}
    .b-gold{background:rgba(201,168,76,.15);color:${GOLD};border:1px solid rgba(201,168,76,.3);}
    .b-green{background:rgba(76,201,138,.15);color:#4CC98A;border:1px solid rgba(76,201,138,.3);}
    .b-red{background:rgba(201,76,76,.15);color:#C94C4C;border:1px solid rgba(201,76,76,.3);}
    .b-blue{background:rgba(76,154,201,.15);color:#4C9AC9;border:1px solid rgba(76,154,201,.3);}
    table{width:100%;border-collapse:collapse;}
    th{text-align:left;padding:12px 16px;font-family:'Cinzel',serif;font-size:11px;letter-spacing:2px;color:${GOLD};border-bottom:1px solid #222;text-transform:uppercase;}
    td{padding:13px 16px;border-bottom:1px solid #181818;font-size:14px;color:${WHITE};}
    tr:hover td{background:rgba(201,168,76,.02);}
    .nav-link{display:flex;align-items:center;gap:12px;padding:10px 14px;border-radius:6px;cursor:pointer;transition:all .2s;font-size:14px;font-weight:500;letter-spacing:.5px;color:${GRAY};border:1px solid transparent;margin-bottom:2px;}
    .nav-link:hover{color:${WHITE};background:${BLACK3};}
    .nav-link.active{color:${GOLD};background:rgba(201,168,76,.08);border-color:rgba(201,168,76,.2);}
    .overlay{position:fixed;inset:0;background:rgba(0,0,0,.85);display:flex;align-items:center;justify-content:center;z-index:1000;backdrop-filter:blur(4px);}
    .modal{background:${BLACK2};border:1px solid #333;border-top:2px solid ${GOLD};border-radius:8px;padding:28px;width:480px;max-width:92vw;animation:fadeIn .25s ease;}
    .lbl{font-size:11px;color:${GOLD};letter-spacing:1.5px;text-transform:uppercase;display:block;margin-bottom:6px;font-weight:600;}
  `}</style>
);

// ─── HELPERS ─────────────────────────────────────────────────────────────────
const Spinner = ({ size = 20 }) => (
  <div className="spin" style={{ width:size, height:size, border:`2px solid #333`, borderTop:`2px solid ${GOLD}`, borderRadius:"50%", display:"inline-block" }} />
);

const Logo = ({ big }) => (
  <div style={{ display:"flex", alignItems:"center", gap:10 }}>
    <div style={{ width:big?48:32, height:big?48:32, background:`linear-gradient(135deg,${GOLD_DARK},${GOLD},${GOLD_LIGHT})`, borderRadius:6, display:"flex", alignItems:"center", justifyContent:"center", fontSize:big?22:14, fontFamily:"'Cinzel',serif", color:BLACK, fontWeight:900, boxShadow:`0 0 20px rgba(201,168,76,.3)` }}>F</div>
    <div>
      <div style={{ fontFamily:"'Cinzel',serif", fontSize:big?28:18, fontWeight:700, background:`linear-gradient(90deg,${GOLD},${GOLD_LIGHT},${GOLD})`, WebkitBackgroundClip:"text", WebkitTextFillColor:"transparent", letterSpacing:3 }}>FALCO CONNECT</div>
      <div style={{ fontSize:big?12:9, color:GRAY, letterSpacing:4, textTransform:"uppercase" }}>HR Management</div>
    </div>
  </div>
);

const Avatar = ({ name, size=32 }) => (
  <div style={{ width:size, height:size, borderRadius:"50%", background:`linear-gradient(135deg,${GOLD_DARK},${GOLD})`, display:"flex", alignItems:"center", justifyContent:"center", fontSize:size*.42, fontWeight:700, color:BLACK, flexShrink:0 }}>
    {(name||"?").charAt(0).toUpperCase()}
  </div>
);

const Field = ({ label, children }) => (
  <div>
    <label className="lbl">{label}</label>
    {children}
  </div>
);

// ─── LOGIN ────────────────────────────────────────────────────────────────────
function LoginPage({ onLogin }) {
  const [email,   setEmail]   = useState("");
  const [pass,    setPass]    = useState("");
  const [err,     setErr]     = useState("");
  const [loading, setLoading] = useState(false);
  const [showP,   setShowP]   = useState(false);

  const login = async () => {
    setErr(""); setLoading(true);
    try {
      const cred = await signInWithEmailAndPassword(auth, email.trim(), pass);
      const snap = await getDoc(doc(db, "users", cred.user.email));
      if (!snap.exists()) throw new Error("User profile not found. Contact Admin.");
      const profile = snap.data();
      if (!profile.active) { await signOut(auth); throw new Error("Account deactivated. Contact Administrator."); }
      onLogin({ uid: cred.user.uid, ...profile });
    } catch (e) {
      const msg = e.code === "auth/invalid-credential" ? "Invalid email or password."
                : e.code === "auth/too-many-requests"  ? "Too many attempts. Try later."
                : e.code === "auth/network-request-failed" ? "No internet connection."
                : e.message;
      setErr(msg);
    }
    setLoading(false);
  };

  return (
    <div style={{ minHeight:"100vh", background:BLACK, display:"flex", alignItems:"center", justifyContent:"center" }}>
      <div style={{ width:420, padding:20 }}>
        <div style={{ textAlign:"center", marginBottom:40 }}><Logo big /></div>
        <div style={{ background:BLACK2, border:`1px solid #333`, borderTop:`2px solid ${GOLD}`, borderRadius:10, padding:36, boxShadow:`0 24px 60px rgba(0,0,0,.6)` }}>
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:20, color:WHITE, marginBottom:6 }}>Welcome Back</div>
          <div style={{ fontSize:14, color:GRAY, marginBottom:28 }}>Sign in to your FALCO CONNECT account</div>

          <div style={{ display:"flex", flexDirection:"column", gap:16 }}>
            <div>
              <label className="lbl">Email Address</label>
              <input
                style={{ background:BLACK3, border:`1px solid #444`, color:WHITE, padding:"11px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:15, width:"100%", outline:"none", display:"block" }}
                type="email"
                placeholder="you@falcoconnect.com"
                value={email}
                onChange={e => setEmail(e.target.value)}
                onKeyDown={e => e.key==="Enter" && login()}
              />
            </div>
            <div>
              <label className="lbl">Password</label>
              <div style={{ position:"relative" }}>
                <input
                  style={{ background:BLACK3, border:`1px solid #444`, color:WHITE, padding:"11px 14px", paddingRight:44, borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:15, width:"100%", outline:"none", display:"block" }}
                  type={showP ? "text" : "password"}
                  placeholder="••••••••"
                  value={pass}
                  onChange={e => setPass(e.target.value)}
                  onKeyDown={e => e.key==="Enter" && login()}
                />
                <button onClick={()=>setShowP(!showP)} style={{ position:"absolute", right:12, top:"50%", transform:"translateY(-50%)", background:"none", border:"none", color:GRAY, cursor:"pointer", fontSize:16, zIndex:10 }}>
                  {showP ? "🙈" : "👁"}
                </button>
              </div>
            </div>

            {err && (
              <div style={{ background:"rgba(201,76,76,.1)", border:"1px solid rgba(201,76,76,.3)", borderRadius:4, padding:"10px 14px", fontSize:13, color:"#C94C4C" }}>
                ⚠ {err}
              </div>
            )}

            <button className="gold-btn" onClick={login} disabled={loading} style={{ width:"100%", padding:13, fontSize:16, marginTop:6 }}>
              {loading ? <span style={{display:"flex",alignItems:"center",justifyContent:"center",gap:10}}><Spinner size={16}/>Signing in...</span> : "Sign In →"}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}


// ─── TOP BAR ─────────────────────────────────────────────────────────────────
function TopBar({ user, onLogout, page }) {
  const [time, setTime] = useState(new Date().toLocaleTimeString("en-PK",{hour:"2-digit",minute:"2-digit"}));
  const [todayRec, setTodayRec] = useState(null);
  const [checking, setChecking] = useState(false);
  const todayStr = new Date().toISOString().slice(0,10);

  useEffect(() => {
    const t = setInterval(() => setTime(new Date().toLocaleTimeString("en-PK",{hour:"2-digit",minute:"2-digit"})), 30000);
    return () => clearInterval(t);
  }, []);

  useEffect(() => {
    const id = `${user.email}_${todayStr}`;
    const unsub = onSnapshot(doc(db,"attendance",id), snap => {
      setTodayRec(snap.exists() ? snap.data() : null);
    });
    return unsub;
  }, [user.email]);

  const checkIn = async () => {
    setChecking(true);
    const now = new Date();
    const id = `${user.email}_${todayStr}`;
    await setDoc(doc(db,"attendance",id), {
      employeeEmail: user.email,
      employeeName: user.name,
      date: todayStr,
      checkIn: now.toLocaleTimeString("en-PK",{hour:"2-digit",minute:"2-digit"}),
      checkOut: null,
      status: "present",
      hoursWorked: null,
      createdAt: serverTimestamp(),
    });
    setChecking(false);
  };

  const checkOut = async () => {
    setChecking(true);
    const now = new Date();
    const [inH, inM] = (todayRec.checkIn||"00:00").split(":").map(Number);
    const hours = ((now.getHours()*60+now.getMinutes()) - (inH*60+inM)) / 60;
    await updateDoc(doc(db,"attendance",`${user.email}_${todayStr}`), {
      checkOut: now.toLocaleTimeString("en-PK",{hour:"2-digit",minute:"2-digit"}),
      hoursWorked: hours.toFixed(1),
    });
    setChecking(false);
  };

  return (
    <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", paddingBottom:16, marginBottom:24, borderBottom:`1px solid #1E1E1E`, flexWrap:"wrap", gap:10 }}>
      {/* Left: breadcrumb */}
      <div style={{ fontSize:13, color:GRAY, letterSpacing:1 }}>
        FALCO CONNECT / <span style={{ color:GOLD }}>{(page||"dashboard").toUpperCase()}</span>
      </div>

      {/* Right: actions */}
      <div style={{ display:"flex", alignItems:"center", gap:10, flexWrap:"wrap" }}>
        {/* Time */}
        <div style={{ fontSize:13, color:GRAY, minWidth:60 }}>{time}</div>

        {/* Check In / Out */}
        {!todayRec ? (
          <button onClick={checkIn} disabled={checking} style={{
            background:"rgba(76,201,138,.1)", border:"1px solid rgba(76,201,138,.4)",
            color:"#4CC98A", padding:"6px 14px", borderRadius:4, cursor:"pointer",
            fontFamily:"'Rajdhani',sans-serif", fontSize:13, fontWeight:600,
            display:"flex", alignItems:"center", gap:6,
          }}>
            {checking ? <Spinner size={12}/> : "✅ Check In"}
          </button>
        ) : !todayRec.checkOut ? (
          <div style={{ display:"flex", alignItems:"center", gap:6 }}>
            <span style={{ fontSize:12, color:"#4CC98A" }}>✅ {todayRec.checkIn}</span>
            <button onClick={checkOut} disabled={checking} style={{
              background:"rgba(201,76,76,.1)", border:"1px solid rgba(201,76,76,.4)",
              color:"#C94C4C", padding:"6px 14px", borderRadius:4, cursor:"pointer",
              fontFamily:"'Rajdhani',sans-serif", fontSize:13, fontWeight:600,
              display:"flex", alignItems:"center", gap:6,
            }}>
              {checking ? <Spinner size={12}/> : "🚪 Out"}
            </button>
          </div>
        ) : (
          <div style={{ fontSize:12, color:GOLD, background:"rgba(201,168,76,.08)", border:"1px solid rgba(201,168,76,.2)", padding:"6px 12px", borderRadius:4 }}>
            ⏱ {todayRec.checkIn}→{todayRec.checkOut} ({todayRec.hoursWorked}h)
          </div>
        )}

        {/* Role badge */}
        <span className={`badge ${user.role==="super_admin"||user.role==="ceo"?"b-gold":user.role==="manager"||user.role==="hr_manager"?"b-blue":"b-green"}`}>
          {user.role==="super_admin" ? "♛ Super Admin" : ROLES.find(r=>r.value===user.role)?.label||user.role}
        </span>

        {/* Logout */}
        <button onClick={onLogout} style={{
          background:"rgba(201,76,76,.08)", border:"1px solid rgba(201,76,76,.3)",
          color:"#C94C4C", padding:"6px 14px", borderRadius:4, cursor:"pointer",
          fontFamily:"'Rajdhani',sans-serif", fontSize:13, fontWeight:700,
          letterSpacing:0.5,
        }}>
          Logout 🚪
        </button>
      </div>
    </div>
  );
}

// ─── SIDEBAR ─────────────────────────────────────────────────────────────────
function Sidebar({ user, page, setPage, onLogout }) {
  const roleObj = ROLES.find(r => r.value === user.role) || ROLES[3];
  const ALL_NAV = [
    { id:"dashboard",   label:"Dashboard",      icon:"⊞" },
    { id:"employees",   label:"Employees",      icon:"👥" },
    { id:"departments", label:"Departments",    icon:"🏢" },
    { id:"attendance",  label:"Attendance",     icon:"📋" },
    { id:"payroll",     label:"Payroll",        icon:"💰" },
    { id:"leaves",      label:"Leave Mgmt",     icon:"🌿" },
    { id:"sales",       label:"Sales & Commission", icon:"💹" },
    { id:"leaderboard", label:"Leaderboard",           icon:"🏆" },
    { id:"settings",    label:"Settings",       icon:"⚙️" },
  ];
  const nav = ALL_NAV.filter(n => roleObj.pages.includes(n.id));

  return (
    <div style={{ width:230, minHeight:"100vh", background:BLACK2, borderRight:`1px solid #1E1E1E`, display:"flex", flexDirection:"column", position:"fixed", left:0, top:0, bottom:0, zIndex:100 }}>
      <div style={{ padding:"24px 20px", borderBottom:"1px solid #1E1E1E" }}><Logo /></div>
      <div style={{ padding:"16px 20px", borderBottom:"1px solid #1E1E1E" }}>
        <div style={{ display:"flex", alignItems:"center", gap:10 }}>
          <Avatar name={user.name} />
          <div style={{ flex:1, minWidth:0 }}>
            <div style={{ fontSize:13, fontWeight:600, color:WHITE, whiteSpace:"nowrap", overflow:"hidden", textOverflow:"ellipsis" }}>{user.name}</div>
            <div style={{ fontSize:11, color:GOLD }}>{user.role==="super_admin" ? "♛ Super Admin" : roleObj.label}</div>
          </div>
        </div>
      </div>
      <nav style={{ flex:1, padding:"12px", overflowY:"auto" }}>
        {nav.map(n => (
          <div key={n.id} className={`nav-link${page===n.id?" active":""}`} onClick={()=>setPage(n.id)}>
            <span style={{ width:18, textAlign:"center" }}>{n.icon}</span>
            <span>{n.label}</span>
          </div>
        ))}
      </nav>
    </div>
  );
}

// ─── DASHBOARD ────────────────────────────────────────────────────────────────
function Dashboard({ user, users, depts }) {
  const activeU = users.filter(u=>u.active).length;
  const totalE  = depts.reduce((a,d)=>a+Number(d.employees||0),0);
  const [onLeaveCount, setOnLeaveCount] = useState(0);
  const [presentToday, setPresentToday] = useState(0);
  const [absentToday, setAbsentToday] = useState(0);
  const [ncnsCount, setNcnsCount] = useState(0);
  const [employees, setEmployees] = useState([]);
  const [todayAttendance, setTodayAttendance] = useState([]);
  const [allLeaves, setAllLeaves] = useState([]);
  const [selectedDept, setSelectedDept] = useState("all");
  const todayStr = new Date().toISOString().slice(0,10);

  useEffect(()=>{
    const unsub = onSnapshot(collection(db,"leaves"), snap=>{
      const docs = snap.docs.map(d=>d.data());
      setAllLeaves(docs);
      const today = docs.filter(l=>
        l.status==="approved" && l.fromDate<=todayStr && l.toDate>=todayStr
      ).length;
      setOnLeaveCount(today);
    });
    return unsub;
  },[]);

  useEffect(()=>{
    const unsub = onSnapshot(collection(db,"attendance"), snap=>{
      const allRecs = snap.docs.map(d=>d.data());
      const todayRecs = allRecs.filter(r=>r.date===todayStr);
      setTodayAttendance(todayRecs);
      setPresentToday(todayRecs.filter(r=>r.status==="present"||r.status==="half_day").length);
      setAbsentToday(todayRecs.filter(r=>r.status==="absent").length);
      const allNcns = allRecs.filter(r=>r.status==="ncns").length;
      setNcnsCount(allNcns);
    });
    return unsub;
  },[]);

  useEffect(()=>{
    const unsub = onSnapshot(collection(db,"employees"), snap=>{
      setEmployees(snap.docs.map(d=>({...d.data(),id:d.id})));
    });
    return unsub;
  },[]);

  // Filter employees by selected department
  const deptFilteredEmployees = selectedDept==="all" ? employees : employees.filter(e=>e.department===selectedDept);

  // Get attendance status for an employee today
  const getEmpStatus = (email) => {
    const rec = todayAttendance.find(r=>r.employeeEmail===email);
    const onLeave = allLeaves.some(l=>l.employeeEmail===email&&l.status==="approved"&&l.fromDate<=todayStr&&l.toDate>=todayStr);
    if (onLeave) return { status:"leave", label:"🌿 On Leave", color:"#4C9AC9", rec:null };
    if (!rec) return { status:"not_marked", label:"⏳ Not Checked In", color:"#888", rec:null };
    if (rec.status==="present") return { status:"present", label:`✅ Present (${rec.checkIn||"—"})`, color:"#4CC98A", rec };
    if (rec.status==="half_day") return { status:"half_day", label:`½ Half Day (${rec.checkIn||"—"})`, color:"#C9A84C", rec };
    if (rec.status==="ncns") return { status:"ncns", label:"🚫 NCNS", color:"#C94C4C", rec };
    if (rec.status==="absent") return { status:"absent", label:"❌ Absent", color:"#C94C4C", rec };
    return { status:"unknown", label:"—", color:"#888", rec };
  };

  // Stats for selected department
  const deptStats = {
    present: deptFilteredEmployees.filter(e=>getEmpStatus(e.email||e.id).status==="present"||getEmpStatus(e.email||e.id).status==="half_day").length,
    absent:  deptFilteredEmployees.filter(e=>getEmpStatus(e.email||e.id).status==="absent"||getEmpStatus(e.email||e.id).status==="ncns").length,
    leave:   deptFilteredEmployees.filter(e=>getEmpStatus(e.email||e.id).status==="leave").length,
    notMarked: deptFilteredEmployees.filter(e=>getEmpStatus(e.email||e.id).status==="not_marked").length,
  };

  const Stat = ({label,val,sub,icon,c=GOLD}) => (
    <div className="card hov" style={{ flex:1, minWidth:150 }}>
      <div style={{ display:"flex", justifyContent:"space-between", alignItems:"flex-start" }}>
        <div>
          <div style={{ fontSize:12, color:GRAY, letterSpacing:1, textTransform:"uppercase", marginBottom:8 }}>{label}</div>
          <div style={{ fontSize:30, fontFamily:"'Cinzel',serif", color:c, fontWeight:700 }}>{val}</div>
          {sub && <div style={{ fontSize:12, color:GRAY, marginTop:4 }}>{sub}</div>}
        </div>
        <div style={{ fontSize:26, opacity:.5 }}>{icon}</div>
      </div>
    </div>
  );

  // Employee dashboard — sirf apna data
  if (user.role === "employee") {
    return (
      <div className="fade">
        <div style={{ marginBottom:28 }}>
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE }}>Good day, <span style={{ color:GOLD }}>{(user.name||"").split(" ")[0]}</span></div>
          <div style={{ color:GRAY, fontSize:14, marginTop:4 }}>{new Date().toLocaleDateString("en-PK",{weekday:"long",year:"numeric",month:"long",day:"numeric"})}</div>
        </div>
        <div style={{ display:"flex", gap:16, flexWrap:"wrap", marginBottom:28 }}>
          <Stat label="My Department" val={user.department||"—"} sub="Your team" icon="🏢" c="#4C9AC9"/>
          <Stat label="Leave Balance" val="12" sub="Days remaining" icon="🌿" c="#4CC98A"/>
          <Stat label="Attendance" val="96%" sub="This month" icon="📋" c={GOLD}/>
          <Stat label="My Salary" val="—" sub="Check payroll" icon="💰" c="#C94C9A"/>
        </div>
        <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:20 }}>
          <div className="card">
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1, marginBottom:18 }}>MY PROFILE</div>
            {[
              ["Full Name", user.name||"—"],
              ["Email", user.email||"—"],
              ["Department", user.department||"—"],
              ["Role", "Employee"],
              ["Status", "🟢 Active"],
            ].map(([k,v])=>(
              <div key={k} style={{ display:"flex", justifyContent:"space-between", padding:"10px 0", borderBottom:"1px solid #1a1a1a" }}>
                <span style={{ fontSize:13, color:GRAY }}>{k}</span>
                <span style={{ fontSize:13, color:WHITE, fontWeight:500 }}>{v}</span>
              </div>
            ))}
          </div>
          <div className="card">
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1, marginBottom:18 }}>QUICK ACTIONS</div>
            {[
              ["🌿", "Apply for Leave", "leaves"],
              ["📋", "View Attendance", "attendance"],
              ["💰", "View Payroll", "payroll"],
              ["⚙️", "Settings", "settings"],
            ].map(([icon, label, pg])=>(
              <div key={label} style={{ display:"flex", alignItems:"center", gap:12, padding:"12px 0", borderBottom:"1px solid #1a1a1a", cursor:"pointer" }}
                onClick={()=> {}}>
                <span style={{ fontSize:20 }}>{icon}</span>
                <span style={{ fontSize:14, color:WHITE }}>{label}</span>
                <span style={{ marginLeft:"auto", color:GOLD }}>→</span>
              </div>
            ))}
          </div>
        </div>
      </div>
    );
  }

  // Admin/Manager dashboard
  return (
    <div className="fade">
      <div style={{ marginBottom:24 }}>
        <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE }}>Good day, <span style={{ color:GOLD }}>{(user.name||"").split(" ")[0]}</span></div>
        <div style={{ color:GRAY, fontSize:14, marginTop:4 }}>{new Date().toLocaleDateString("en-PK",{weekday:"long",year:"numeric",month:"long",day:"numeric"})}</div>
      </div>
      <div style={{ display:"flex", gap:16, flexWrap:"wrap", marginBottom:16 }}>
        <Stat label="Total Employees" val={totalE}       sub="Across all departments" icon="👥" c={GOLD}     />
        <Stat label="Departments"     val={depts.length} sub="Active"                 icon="🏢" c="#4C9AC9"  />
        <Stat label="Active Users"    val={activeU}      sub={`of ${users.length} total`} icon="✅" c="#4CC98A"/>
      </div>
      <div style={{ display:"flex", gap:16, flexWrap:"wrap", marginBottom:24 }}>
        <Stat label="Present Today"   val={presentToday}  sub="Checked in today"       icon="✅" c="#4CC98A"  />
        <Stat label="Absent Today"    val={absentToday}   sub="No show, no leave"      icon="❌" c="#C94C4C"  />
        <Stat label="NCNS (Total)"    val={ncnsCount}      sub="All-time warnings"      icon="🚫" c="#C94C4C"  />
        <Stat label="On Leave Today"  val={onLeaveCount}  sub="Approved leaves today"  icon="🌿" c="#C94C9A"  />
      </div>

      {/* Department Filter Pills */}
      <div style={{ display:"flex", gap:8, flexWrap:"wrap", marginBottom:20 }}>
        <button onClick={()=>setSelectedDept("all")} style={{
          background: selectedDept==="all" ? GOLD : "transparent",
          color: selectedDept==="all" ? BLACK : GRAY,
          border:`1px solid ${selectedDept==="all"?GOLD:"#333"}`,
          padding:"7px 18px", borderRadius:20, cursor:"pointer",
          fontFamily:"'Rajdhani',sans-serif", fontSize:13, fontWeight:700, transition:"all .2s",
        }}>All ({employees.length})</button>
        {depts.map(d=>(
          <button key={d.id} onClick={()=>setSelectedDept(d.name)} style={{
            background: selectedDept===d.name ? (d.color||GOLD) : "transparent",
            color: selectedDept===d.name ? BLACK : GRAY,
            border:`1px solid ${selectedDept===d.name?(d.color||GOLD):"#333"}`,
            padding:"7px 18px", borderRadius:20, cursor:"pointer",
            fontFamily:"'Rajdhani',sans-serif", fontSize:13, fontWeight:700, transition:"all .2s",
          }}>{d.name} ({employees.filter(e=>e.department===d.name).length})</button>
        ))}
      </div>

      {/* Live Attendance for Selected Department */}
      <div className="card" style={{ marginBottom:24, padding:0, overflow:"hidden" }}>
        <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", padding:"14px 18px", borderBottom:"1px solid #222" }}>
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1 }}>
            {selectedDept==="all"?"ALL DEPARTMENTS":selectedDept.toUpperCase()} — TODAY'S ATTENDANCE
          </div>
          <div style={{ display:"flex", gap:14, fontSize:12 }}>
            <span style={{color:"#4CC98A"}}>✅ {deptStats.present}</span>
            <span style={{color:"#C94C4C"}}>❌ {deptStats.absent}</span>
            <span style={{color:"#4C9AC9"}}>🌿 {deptStats.leave}</span>
            <span style={{color:"#888"}}>⏳ {deptStats.notMarked}</span>
          </div>
        </div>
        <div style={{ maxHeight:380, overflowY:"auto" }}>
          <table>
            <thead><tr><th>Employee</th><th>Designation</th><th>Status</th><th>Check In</th><th>Check Out</th></tr></thead>
            <tbody>
              {deptFilteredEmployees.map(e=>{
                const email = e.email||e.id;
                const st = getEmpStatus(email);
                return (
                  <tr key={e.id}>
                    <td><div style={{display:"flex",alignItems:"center",gap:8}}><Avatar name={e.name} size={26}/><span style={{fontSize:13,fontWeight:600}}>{e.name}</span></div></td>
                    <td style={{color:GRAY,fontSize:13}}>{e.designation||"—"}</td>
                    <td><span style={{fontSize:12,color:st.color,fontWeight:600}}>{st.label}</span></td>
                    <td style={{fontSize:12,color:"#4CC98A"}}>{st.rec?.checkIn||"—"}</td>
                    <td style={{fontSize:12,color:"#C94C4C"}}>{st.rec?.checkOut||"—"}</td>
                  </tr>
                );
              })}
              {deptFilteredEmployees.length===0&&<tr><td colSpan={5} style={{textAlign:"center",color:GRAY,padding:24}}>No employees in this department.</td></tr>}
            </tbody>
          </table>
        </div>
      </div>

      <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:20 }}>
        <div className="card">
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1, marginBottom:18 }}>DEPARTMENTS</div>
          {depts.length===0 && <div style={{color:GRAY,fontSize:13}}>No departments yet.</div>}
          {depts.map(d=>(
            <div key={d.id} style={{ display:"flex", alignItems:"center", gap:12, marginBottom:14 }}>
              <div style={{ width:10, height:10, borderRadius:"50%", background:d.color||GOLD, flexShrink:0 }}/>
              <div style={{ flex:1 }}>
                <div style={{ fontSize:14, color:WHITE, fontWeight:500 }}>{d.name}</div>
                <div style={{ fontSize:12, color:GRAY }}>{d.head||"No head assigned"}</div>
              </div>
              <div style={{ fontSize:13, color:GOLD, fontWeight:600 }}>{d.employees||0}</div>
            </div>
          ))}
        </div>
        <div className="card">
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1, marginBottom:18 }}>SYSTEM INFO</div>
          {[
            ["Firebase Project","falco-connect-cc992"],
            ["Your Role", ROLES.find(r=>r.value===user.role)?.label||user.role],
            ["Your Department", user.department||"—"],
            ["Total Users", users.length],
            ["Status","🟢 Connected"],
          ].map(([k,v])=>(
            <div key={k} style={{ display:"flex", justifyContent:"space-between", padding:"10px 0", borderBottom:"1px solid #1a1a1a" }}>
              <span style={{ fontSize:13, color:GRAY }}>{k}</span>
              <span style={{ fontSize:13, color:WHITE, fontWeight:500 }}>{v}</span>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

// ─── USERS PAGE ───────────────────────────────────────────────────────────────
function UsersPage({ users, setUsers }) {
  const [search,  setSearch]  = useState("");
  const [modal,   setModal]   = useState(false);
  const [editing, setEditing] = useState(null);
  const [form,    setForm]    = useState({ name:"", email:"", password:"", role:"employee", department:"", active:true });
  const [saving,  setSaving]  = useState(false);
  const [err,     setErr]     = useState("");

  const filtered = users.filter(u =>
    (u.name||"").toLowerCase().includes(search.toLowerCase()) ||
    (u.email||"").toLowerCase().includes(search.toLowerCase())
  );

  const openAdd  = () => { setEditing(null); setForm({name:"",email:"",password:"",role:"employee",department:"",active:true}); setErr(""); setModal(true); };
  const openEdit = u  => { setEditing(u); setForm({...u, password:""}); setErr(""); setModal(true); };

  const save = async () => {
    if (!form.name||!form.email) { setErr("Name and email are required."); return; }
    setSaving(true); setErr("");
    try {
      if (editing) {
        await updateDoc(doc(db,"users",editing.email), { name:form.name, role:form.role, department:form.department, active:form.active });
        setUsers(users.map(u => u.email===editing.email ? {...u,...form} : u));
      } else {
        const cred = await createUserWithEmailAndPassword(auth, form.email, form.password||"falco-connect123");
        await setDoc(doc(db,"users",form.email), { name:form.name, email:form.email, role:form.role, department:form.department, active:form.active, createdAt:serverTimestamp() });
        setUsers([...users, { uid:cred.user.uid, ...form }]);
      }
      setModal(false);
    } catch(e) {
      setErr(e.code==="auth/email-already-in-use" ? "Email already exists." : e.message);
    }
    setSaving(false);
  };

  const toggleActive = async u => {
    await updateDoc(doc(db,"users",u.email), { active:!u.active });
    setUsers(users.map(x => x.email===u.email ? {...x, active:!x.active} : x));
  };

  const del = async u => {
    if (!window.confirm(`Delete ${u.name}?`)) return;
    await deleteDoc(doc(db,"users",u.email));
    setUsers(users.filter(x => x.email!==u.email));
  };

  return (
    <div className="fade">
      <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:24 }}>
        <div>
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE }}>Users & Access Control</div>
          <div style={{ color:GRAY, fontSize:14, marginTop:4 }}>Changes saved to Firebase instantly</div>
        </div>
        <button className="gold-btn" onClick={openAdd}>+ Add User</button>
      </div>

      <div style={{ marginBottom:18, position:"relative", maxWidth:340 }}>
        <span style={{ position:"absolute", left:12, top:"50%", transform:"translateY(-50%)", color:GRAY, pointerEvents:"none" }}>⌕</span>
        <input className="drv-input" placeholder="Search users..." value={search} onChange={e=>setSearch(e.target.value)} style={{ paddingLeft:36 }}/>
      </div>

      <div className="card" style={{ padding:0, overflow:"hidden" }}>
        <table>
          <thead><tr><th>Name</th><th>Email</th><th>Role</th><th>Department</th><th>Status</th><th>Actions</th></tr></thead>
          <tbody>
            {filtered.map(u=>(
              <tr key={u.email}>
                <td>
                  <div style={{ display:"flex", alignItems:"center", gap:10 }}>
                    <Avatar name={u.name} size={30}/>
                    <span style={{ fontWeight:500 }}>{u.name}</span>
                    {u.role==="super_admin" && <span>♛</span>}
                  </div>
                </td>
                <td style={{ color:GRAY, fontSize:13 }}>{u.email}</td>
                <td><span className={`badge ${u.role==="super_admin"?"b-gold":"b-blue"}`}>{ROLES.find(r=>r.value===u.role)?.label||u.role}</span></td>
                <td style={{ color:GRAY, fontSize:13 }}>{u.department||"—"}</td>
                <td><span className={`badge ${u.active?"b-green":"b-red"}`}>{u.active?"Active":"Inactive"}</span></td>
                <td>
                  <div style={{ display:"flex", gap:6 }}>
                    <button className="ghost-btn" onClick={()=>openEdit(u)} style={{ padding:"5px 10px", fontSize:12 }}>Edit</button>
                    {u.role!=="super_admin" && <>
                      <button className="ghost-btn" onClick={()=>toggleActive(u)} style={{ padding:"5px 10px", fontSize:12, borderColor:u.active?"#666":"#4CC98A", color:u.active?"#666":"#4CC98A" }}>
                        {u.active?"Deactivate":"Activate"}
                      </button>
                      <button className="danger-btn" onClick={()=>del(u)}>✕</button>
                    </>}
                  </div>
                </td>
              </tr>
            ))}
            {filtered.length===0 && <tr><td colSpan={6} style={{ textAlign:"center", color:GRAY, padding:32 }}>No users found.</td></tr>}
          </tbody>
        </table>
      </div>

      {modal && (
        <div className="overlay" onClick={()=>setModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()}>
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:18, color:GOLD, marginBottom:22 }}>{editing?"Edit User":"New User"}</div>
            <div style={{ display:"flex", flexDirection:"column", gap:14 }}>
              <Field label="Full Name"><input className="drv-input" value={form.name} onChange={e=>setForm({...form,name:e.target.value})} placeholder="John Doe"/></Field>
              {!editing && <Field label="Email"><input className="drv-input" value={form.email} onChange={e=>setForm({...form,email:e.target.value})} placeholder="john@falco-connect.com"/></Field>}
              <Field label={editing?"New Password (blank = no change)":"Password (min 6 chars)"}>
                <input className="drv-input" type="password" value={form.password} onChange={e=>setForm({...form,password:e.target.value})} placeholder="••••••••"/>
              </Field>
              <Field label="Role">
                <select className="drv-select" value={form.role} onChange={e=>setForm({...form,role:e.target.value})}>
                  {ROLES.map(r=><option key={r.value} value={r.value}>{r.label}</option>)}
                </select>
              </Field>
              <Field label="Department">
                <input className="drv-input" value={form.department} onChange={e=>setForm({...form,department:e.target.value})} placeholder="e.g. IT"/>
              </Field>
              <div style={{ display:"flex", alignItems:"center", gap:10 }}>
                <input type="checkbox" id="active" checked={form.active} onChange={e=>setForm({...form,active:e.target.checked})} style={{ width:"auto", accentColor:GOLD }}/>
                <label htmlFor="active" style={{ color:WHITE, fontSize:14, cursor:"pointer" }}>Account Active</label>
              </div>
              {err && <div style={{ background:"rgba(201,76,76,.1)", border:"1px solid rgba(201,76,76,.3)", borderRadius:4, padding:"10px 14px", fontSize:13, color:"#C94C4C" }}>⚠ {err}</div>}
              <div style={{ display:"flex", gap:12, marginTop:6 }}>
                <button className="gold-btn" onClick={save} disabled={saving} style={{ flex:1 }}>
                  {saving ? <Spinner size={14}/> : (editing?"Save Changes":"Create User")}
                </button>
                <button className="ghost-btn" onClick={()=>setModal(false)} style={{ flex:1 }}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

// ─── DEPARTMENTS ──────────────────────────────────────────────────────────────
function DeptsPage({ depts, setDepts }) {
  const [modal,   setModal]   = useState(false);
  const [editing, setEditing] = useState(null);
  const [form,    setForm]    = useState({ name:"", head:"", employees:0, color:GOLD });
  const [saving,  setSaving]  = useState(false);

  const openAdd  = () => { setEditing(null); setForm({name:"",head:"",employees:0,color:GOLD}); setModal(true); };
  const openEdit = d  => { setEditing(d); setForm({...d}); setModal(true); };

  const save = async () => {
    if (!form.name) return;
    setSaving(true);
    const id = editing?.id || Date.now().toString();
    await setDoc(doc(db,"departments",id), {...form, id});
    if (editing) setDepts(depts.map(d => d.id===editing.id ? {...form,id} : d));
    else         setDepts([...depts, {...form,id}]);
    setSaving(false); setModal(false);
  };

  const del = async d => {
    if (!window.confirm(`Delete ${d.name}?`)) return;
    await deleteDoc(doc(db,"departments",d.id));
    setDepts(depts.filter(x => x.id!==d.id));
  };

  return (
    <div className="fade">
      <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:28 }}>
        <div>
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE }}>Departments</div>
          <div style={{ color:GRAY, fontSize:14, marginTop:4 }}>Saved in real-time to Firebase</div>
        </div>
        <button className="gold-btn" onClick={openAdd}>+ New Department</button>
      </div>

      <div style={{ display:"grid", gridTemplateColumns:"repeat(auto-fill,minmax(230px,1fr))", gap:16 }}>
        {depts.map(d=>(
          <div key={d.id} className="card hov" style={{ borderTop:`3px solid ${d.color||GOLD}` }}>
            <div style={{ display:"flex", justifyContent:"space-between", alignItems:"flex-start", marginBottom:14 }}>
              <div style={{ width:42, height:42, borderRadius:8, background:`${d.color||GOLD}22`, border:`1px solid ${d.color||GOLD}44`, display:"flex", alignItems:"center", justifyContent:"center", fontSize:20 }}>🏢</div>
              <div style={{ display:"flex", gap:6 }}>
                <button className="ghost-btn" onClick={()=>openEdit(d)} style={{ padding:"4px 10px", fontSize:11 }}>Edit</button>
                <button className="danger-btn" onClick={()=>del(d)} style={{ padding:"4px 8px" }}>✕</button>
              </div>
            </div>
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:15, color:WHITE, marginBottom:6 }}>{d.name}</div>
            <div style={{ fontSize:13, color:GRAY, marginBottom:12 }}>Head: <span style={{ color:d.head?WHITE:GRAY }}>{d.head||"—"}</span></div>
            <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center" }}>
              <span style={{ fontSize:12, color:GRAY }}>Employees</span>
              <span style={{ fontSize:24, fontFamily:"'Cinzel',serif", color:d.color||GOLD, fontWeight:700 }}>{d.employees||0}</span>
            </div>
          </div>
        ))}
        {depts.length===0 && <div style={{ color:GRAY, fontSize:14 }}>No departments yet. Add one!</div>}
      </div>

      {modal && (
        <div className="overlay" onClick={()=>setModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()}>
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:18, color:GOLD, marginBottom:22 }}>{editing?"Edit Department":"New Department"}</div>
            <div style={{ display:"flex", flexDirection:"column", gap:14 }}>
              <Field label="Name"><input className="drv-input" value={form.name} onChange={e=>setForm({...form,name:e.target.value})} placeholder="e.g. Engineering"/></Field>
              <Field label="Head"><input className="drv-input" value={form.head} onChange={e=>setForm({...form,head:e.target.value})} placeholder="Person's name"/></Field>
              <Field label="Employee Count"><input className="drv-input" type="number" value={form.employees} onChange={e=>setForm({...form,employees:+e.target.value||0})}/></Field>
              <Field label="Color">
                <div style={{ display:"flex", gap:10, alignItems:"center" }}>
                  <input type="color" value={form.color} onChange={e=>setForm({...form,color:e.target.value})} style={{ width:50, height:38, padding:2, cursor:"pointer", background:"none", border:"none" }}/>
                  <span style={{ fontSize:13, color:GRAY }}>{form.color}</span>
                </div>
              </Field>
              <div style={{ display:"flex", gap:12, marginTop:6 }}>
                <button className="gold-btn" onClick={save} disabled={saving} style={{ flex:1 }}>{saving?<Spinner size={14}/>:"Save"}</button>
                <button className="ghost-btn" onClick={()=>setModal(false)} style={{ flex:1 }}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

// ─── SETTINGS ────────────────────────────────────────────────────────────────
function SettingsPage({ user }) {
  const [pass,    setPass]    = useState({ newP:"", confirm:"" });
  const [msg,     setMsg]     = useState("");
  const [loading, setLoading] = useState(false);

  const changePass = async () => {
    if (pass.newP!==pass.confirm) { setMsg("❌ Passwords don't match."); return; }
    if (pass.newP.length<6)       { setMsg("❌ Minimum 6 characters."); return; }
    setLoading(true);
    try {
      await updatePassword(auth.currentUser, pass.newP);
      setMsg("✅ Password updated!"); setPass({newP:"",confirm:""});
    } catch(e) { setMsg(`❌ ${e.message}`); }
    setLoading(false);
  };

  return (
    <div className="fade" style={{ maxWidth:560 }}>
      <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE, marginBottom:28 }}>Settings</div>
      <div className="card" style={{ marginBottom:20 }}>
        <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1, marginBottom:18 }}>MY ACCOUNT</div>
        {[["Name",user.name],["Email",user.email],["Role",ROLES.find(r=>r.value===user.role)?.label||user.role],["Department",user.department||"—"]].map(([k,v])=>(
          <div key={k} style={{ display:"flex", justifyContent:"space-between", padding:"10px 0", borderBottom:"1px solid #1a1a1a" }}>
            <span style={{ fontSize:13, color:GRAY }}>{k}</span>
            <span style={{ fontSize:13, color:WHITE }}>{v}</span>
          </div>
        ))}
      </div>
      <div className="card" style={{ marginBottom:20 }}>
        <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1, marginBottom:18 }}>CHANGE PASSWORD</div>
        <div style={{ display:"flex", flexDirection:"column", gap:14 }}>
          <Field label="New Password"><input className="drv-input" type="password" value={pass.newP} onChange={e=>setPass({...pass,newP:e.target.value})} placeholder="••••••••"/></Field>
          <Field label="Confirm Password"><input className="drv-input" type="password" value={pass.confirm} onChange={e=>setPass({...pass,confirm:e.target.value})} placeholder="••••••••"/></Field>
          {msg && <div style={{ fontSize:13, color:msg.startsWith("✅")?"#4CC98A":"#C94C4C" }}>{msg}</div>}
          <button className="gold-btn" onClick={changePass} disabled={loading}>{loading?<Spinner size={14}/>:"Update Password"}</button>
        </div>
      </div>
      <div className="card">
        <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1, marginBottom:12 }}>FIREBASE</div>
        <div style={{ display:"flex", alignItems:"center", gap:10 }}>
          <div style={{ width:10, height:10, borderRadius:"50%", background:"#4CC98A" }}/>
          <span style={{ fontSize:13, color:"#4CC98A" }}>Connected — falco-connect-cc992</span>
        </div>
      </div>
    </div>
  );
}


// ─── EMPLOYEES MODULE (merged with Users & Access) ────────────────────────────
function EmployeesPage({ depts, currentUser }) {
  const [emps,    setEmps]    = useState([]);
  const [loading, setLoading] = useState(true);
  const [search,  setSearch]  = useState("");
  const [filter,  setFilter]  = useState("all");
  const [modal,   setModal]   = useState(false);
  const [viewing, setViewing] = useState(null);
  const [editing, setEditing] = useState(null);
  const [saving,  setSaving]  = useState(false);
  const [err,     setErr]     = useState("");
  const [activeTab, setActiveTab] = useState("employees");
  const [users,   setUsers]   = useState([]);
  const EMPTY = { name:"", fatherHusbandName:"", maritalStatus:"single", email:"", phone:"", cnic:"", department:"", designation:"", basicSalary:"", punctualityBonus:"0", grossSalary:"", joinDate:"", dob:"", nationality:"Pakistani", religion:"", address:"", education:"", joinSource:"walk-in", referralName:"", emergencyContact:"", emergencyRelation:"", status:"active", gender:"male", role:"employee", pcUsername:"", pcPassword:"" };
  const [form, setForm] = useState(EMPTY);
  const canManage = ["super_admin","ceo","manager","hr_manager"].includes(currentUser?.role);

  useEffect(() => {
    const unsub1 = onSnapshot(collection(db,"employees"), snap => {
      setEmps(snap.docs.map(d => ({ ...d.data(), id:d.id })));
      setLoading(false);
    });
    const unsub2 = onSnapshot(collection(db,"users"), snap => {
      setUsers(snap.docs.map(d => ({ ...d.data(), email:d.id,
        name: d.data().name || d.data().NAME || "",
        role: d.data().role || "employee",
        active: d.data().active !== undefined ? d.data().active : true,
      })));
    });
    return () => { unsub1(); unsub2(); };
  }, []);

  const filtered = emps.filter(e => {
    const matchSearch = (e.name||"").toLowerCase().includes(search.toLowerCase()) ||
                        (e.designation||"").toLowerCase().includes(search.toLowerCase()) ||
                        (e.department||"").toLowerCase().includes(search.toLowerCase());
    const matchFilter = filter==="all" || e.status===filter || e.department===filter;
    return matchSearch && matchFilter;
  });

  const openAdd  = () => { setEditing(null); setForm(EMPTY); setErr(""); setModal(true); };
  const openEdit = e  => { setEditing(e); setForm({...e}); setErr(""); setViewing(null); setModal(true); };
  const openView = e  => setViewing(e);

  const save = async () => {
    if (!form.name || !form.department) { setErr("Name and Department are required."); return; }
    if (!editing && !form.email) { setErr("Email is required to create login credentials."); return; }
    if (!editing && !form.cnic) { setErr("CNIC is required — it will be used as the default password."); return; }
    setSaving(true); setErr("");

    const id = editing?.id || Date.now().toString();
    const emailTrimmed = (form.email||"").trim();

    try {
      // Step 1: Save employee record (always, this is the source of truth)
      await setDoc(doc(db,"employees",id), { ...form, id, updatedAt: serverTimestamp() });

      // Step 2: Handle Firebase Auth + user profile
      if (!editing && emailTrimmed && form.cnic) {
        let authCreated = false;
        try {
          await createUserWithEmailAndPassword(auth, emailTrimmed, form.cnic.replace(/-/g,""));
          authCreated = true;
        } catch(authErr) {
          if (authErr.code === "auth/email-already-in-use") {
            authCreated = true; // already exists, that's fine — proceed to profile
          } else {
            throw authErr; // real error (invalid email, weak password etc)
          }
        }

        // Step 3: ALWAYS create/update Firestore user profile — setDoc with merge
        // This runs whether auth was just created OR already existed
        if (authCreated) {
          await setDoc(doc(db,"users",emailTrimmed), {
            name: form.name,
            email: emailTrimmed,
            role: form.role || "employee",
            department: form.department,
            active: true,
            createdAt: serverTimestamp(),
          }, { merge: true }); // merge=true creates if missing, updates if exists
        }
      } else if (editing && emailTrimmed) {
        // Editing — always merge profile so it gets created if it was missing
        await setDoc(doc(db,"users",emailTrimmed), {
          name: form.name,
          email: emailTrimmed,
          department: form.department,
          active: form.status !== "inactive",
        }, { merge: true });
      }

      setModal(false);
    } catch(e) {
      setErr(`Employee saved, but login setup failed: ${e.message}. Use "Fix Login" button to retry.`);
    }
    setSaving(false);
  };

  // Retry/fix login profile for an existing employee (e.g. if it failed before)
  const fixLogin = async (emp) => {
    const email = (emp.email||"").trim();
    if (!email) { alert("This employee has no email set."); return; }
    const correctPassword = (emp.cnic||"").replace(/-/g,"") || "falcoconnect123";

    try {
      // Create/merge user profile — fixes "User profile not found" errors
      await setDoc(doc(db,"users",email), {
        name: emp.name,
        email: email,
        role: emp.role || "employee",
        department: emp.department || "",
        active: true,
        createdAt: serverTimestamp(),
      }, { merge: true });

      // Try creating auth account — if it already exists, password CANNOT be changed from here
      try {
        await createUserWithEmailAndPassword(auth, email, correctPassword);
        alert(`✅ Account created! ${emp.name} can log in with:\nEmail: ${email}\nPassword: ${correctPassword}`);
      } catch(authErr) {
        if (authErr.code === "auth/email-already-in-use") {
          alert(`✅ Profile fixed for ${emp.name}.\n\n⚠️ However, their PASSWORD cannot be changed from here.\nTheir password is whatever CNIC was set to WHEN THE ACCOUNT WAS FIRST CREATED — not their current CNIC.\n\nTo reset it: Firebase Console → Authentication → click "${email}" → ⋮ menu → Reset Password.\n\nCorrect password (current CNIC, no dashes) should be: ${correctPassword}`);
        } else {
          throw authErr;
        }
      }
    } catch(e) {
      alert(`❌ Error: ${e.message}`);
    }
  };

  // Delete auth account + user profile, then recreate fresh — solves password mismatch
  const resetCredentials = async (emp) => {
    const email = (emp.email||"").trim();
    if (!email) { alert("This employee has no email set."); return; }
    const correctPassword = (emp.cnic||"").replace(/-/g,"") || "falcoconnect123";
    if (!window.confirm(`This will reset login for ${emp.name}.\n\nNew password will be: ${correctPassword}\n\nNote: this only works if the OLD account is first deleted from Firebase Console → Authentication.\n\nHave you deleted "${email}" from Authentication already?\n\nClick OK if yes, Cancel if no.`)) return;
    try {
      await createUserWithEmailAndPassword(auth, email, correctPassword);
      await setDoc(doc(db,"users",email), {
        name: emp.name, email, role: emp.role||"employee",
        department: emp.department||"", active: true, createdAt: serverTimestamp(),
      }, { merge: true });
      alert(`✅ New account created!\nEmail: ${email}\nPassword: ${correctPassword}`);
    } catch(e) {
      if (e.code==="auth/email-already-in-use") {
        alert(`❌ Account still exists in Authentication. Please delete "${email}" from Firebase Console → Authentication first, then click this button again.`);
      } else {
        alert(`❌ Error: ${e.message}`);
      }
    }
  };

  const del = async e => {
    if (!window.confirm(`Delete ${e.name}?`)) return;
    await deleteDoc(doc(db,"employees",e.id));
    if (viewing?.id===e.id) setViewing(null);
  };

  const StatusBadge = ({s}) => (
    <span className={`badge ${s==="active"?"b-green":s==="on_leave"?"b-gold":"b-red"}`}>
      {s==="active"?"Active":s==="on_leave"?"On Leave":"Inactive"}
    </span>
  );

  const toggleUserActive = async u => {
    await updateDoc(doc(db,"users",u.email), { active:!u.active });
  };
  const deleteUser = async u => {
    if (!window.confirm(`Delete ${u.name}?`)) return;
    await deleteDoc(doc(db,"users",u.email));
  };
  const changeRole = async (u, role) => {
    await updateDoc(doc(db,"users",u.email), { role });
  };

  return (
    <div className="fade">
      {/* Header */}
      <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:20 }}>
        <div>
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE }}>
            {activeTab==="employees" ? "Employees" : "Users & Access"}
          </div>
          <div style={{ color:GRAY, fontSize:14, marginTop:4 }}>
            {activeTab==="employees" ? `${emps.length} total — adding employee creates login automatically` : `${users.length} total users — manage roles and access`}
          </div>
        </div>
        {canManage && activeTab==="employees" && <button className="gold-btn" onClick={openAdd}>+ Add Employee</button>}
      </div>

      {/* Tabs */}
      <div style={{ display:"flex", gap:4, marginBottom:24, borderBottom:`1px solid #222`, paddingBottom:0 }}>
        {[["employees","👥 Employees"],["users","🔐 Users & Access"]].map(([tab,label])=>(
          <div key={tab} onClick={()=>setActiveTab(tab)} style={{
            padding:"10px 20px", cursor:"pointer", fontSize:14, fontWeight:600,
            color: activeTab===tab ? GOLD : GRAY,
            borderBottom: activeTab===tab ? `2px solid ${GOLD}` : "2px solid transparent",
            transition:"all .2s", letterSpacing:0.5,
          }}>{label}</div>
        ))}
      </div>

      {/* Search & Filter - Employees Tab */}
      {activeTab==="employees" && (
      <div style={{ display:"flex", gap:12, marginBottom:20 }}>
        <div style={{ position:"relative", flex:1, maxWidth:340 }}>
          <span style={{ position:"absolute", left:12, top:"50%", transform:"translateY(-50%)", color:GRAY, pointerEvents:"none" }}>⌕</span>
          <input style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px 10px 36px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
            placeholder="Search by name, role, department..." value={search} onChange={e=>setSearch(e.target.value)}/>
        </div>
        <select style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, outline:"none", cursor:"pointer" }}
          value={filter} onChange={e=>setFilter(e.target.value)}>
          <option value="all">All Departments</option>
          {depts.map(d=><option key={d.id} value={d.name}>{d.name}</option>)}
          <option value="active">Active Only</option>
          <option value="on_leave">On Leave</option>
          <option value="inactive">Inactive</option>
        </select>
      </div>
      )}

      {/* Users & Access Tab */}
      {activeTab==="users" && (
        <div>
          <div style={{ marginBottom:16, position:"relative", maxWidth:340 }}>
            <span style={{ position:"absolute", left:12, top:"50%", transform:"translateY(-50%)", color:GRAY, pointerEvents:"none" }}>⌕</span>
            <input style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px 10px 36px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
              placeholder="Search users..." value={search} onChange={e=>setSearch(e.target.value)}/>
          </div>
          <div className="card" style={{ padding:0, overflow:"hidden" }}>
            <table>
              <thead><tr><th>Name</th><th>Email</th><th>Role</th><th>Department</th><th>Status</th><th>Actions</th></tr></thead>
              <tbody>
                {users.filter(u=>(u.name||"").toLowerCase().includes(search.toLowerCase())||(u.email||"").toLowerCase().includes(search.toLowerCase())).map(u=>(
                  <tr key={u.email}>
                    <td>
                      <div style={{ display:"flex", alignItems:"center", gap:10 }}>
                        <Avatar name={u.name} size={30}/>
                        <span style={{ fontWeight:500 }}>{u.name}</span>
                        {(u.role==="super_admin"||u.role==="ceo") && <span>♛</span>}
                      </div>
                    </td>
                    <td style={{ color:GRAY, fontSize:13 }}>{u.email}</td>
                    <td>
                      {canManage && u.role!=="super_admin" ? (
                        <select style={{ background:BLACK3, border:"1px solid #333", color:WHITE, padding:"4px 8px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:12, outline:"none", cursor:"pointer" }}
                          value={u.role} onChange={e=>changeRole(u,e.target.value)}>
                          {ROLES.map(r=><option key={r.value} value={r.value}>{r.label}</option>)}
                        </select>
                      ) : (
                        <span className={`badge ${u.role==="super_admin"||u.role==="ceo"?"b-gold":u.role==="manager"||u.role==="hr_manager"?"b-blue":"b-green"}`}>
                          {ROLES.find(r=>r.value===u.role)?.label||u.role}
                        </span>
                      )}
                    </td>
                    <td style={{ color:GRAY, fontSize:13 }}>{u.department||"—"}</td>
                    <td>
                      <span className={`badge ${u.active?"b-green":"b-red"}`}>{u.active?"Active":"Inactive"}</span>
                    </td>
                    <td>
                      {u.role!=="super_admin" && canManage && (
                        <div style={{ display:"flex", gap:6 }}>
                          <button className="ghost-btn" onClick={()=>toggleUserActive(u)}
                            style={{ padding:"4px 10px", fontSize:12, borderColor:u.active?"#888":"#4CC98A", color:u.active?"#888":"#4CC98A" }}>
                            {u.active?"Deactivate":"Activate"}
                          </button>
                          <button className="danger-btn" onClick={()=>deleteUser(u)} style={{ padding:"4px 8px" }}>✕</button>
                        </div>
                      )}
                    </td>
                  </tr>
                ))}
                {users.length===0&&<tr><td colSpan={6} style={{ textAlign:"center", color:GRAY, padding:32 }}>No users found.</td></tr>}
              </tbody>
            </table>
          </div>
        </div>
      )}

      <div style={{ display:"flex", gap:20 }}>
        {/* Table */}
        <div style={{ flex:1 }}>
          <div className="card" style={{ padding:0, overflow:"hidden" }}>
            {loading ? (
              <div style={{ display:"flex", justifyContent:"center", padding:40 }}><Spinner size={28}/></div>
            ) : (
              <table>
                <thead><tr><th>Employee</th><th>Department</th><th>Designation</th><th>Phone</th><th>Status</th><th>Actions</th></tr></thead>
                <tbody>
                  {filtered.map(e=>(
                    <tr key={e.id} style={{ cursor:"pointer" }} onClick={()=>openView(e)}>
                      <td>
                        <div style={{ display:"flex", alignItems:"center", gap:10 }}>
                          <Avatar name={e.name} size={32}/>
                          <div>
                            <div style={{ fontWeight:600, fontSize:14 }}>{e.name}</div>
                            <div style={{ fontSize:12, color:GRAY }}>{e.email||"—"}</div>
                          </div>
                        </div>
                      </td>
                      <td style={{ color:GRAY, fontSize:13 }}>{e.department||"—"}</td>
                      <td style={{ fontSize:13 }}>{e.designation||"—"}</td>
                      <td style={{ color:GRAY, fontSize:13 }}>{e.phone||"—"}</td>
                      <td onClick={ev=>ev.stopPropagation()}><StatusBadge s={e.status}/></td>
                      <td onClick={ev=>ev.stopPropagation()}>
                        <div style={{ display:"flex", gap:6 }}>
                          <button className="ghost-btn" onClick={()=>openEdit(e)} style={{ padding:"4px 10px", fontSize:12 }}>Edit</button>
                          {canManage && <button className="ghost-btn" onClick={()=>fixLogin(e)} style={{ padding:"4px 10px", fontSize:12, borderColor:"#4C9AC9", color:"#4C9AC9" }} title="Re-create login if employee can't sign in">🔧 Fix Login</button>}
                          {canManage && <button className="ghost-btn" onClick={()=>resetCredentials(e)} style={{ padding:"4px 10px", fontSize:12, borderColor:"#C9A84C", color:"#C9A84C" }} title="Use after deleting old account from Firebase Auth">🔑 Reset</button>}
                          <button className="danger-btn" onClick={()=>del(e)} style={{ padding:"4px 8px" }}>✕</button>
                        </div>
                      </td>
                    </tr>
                  ))}
                  {filtered.length===0&&!loading && (
                    <tr><td colSpan={6} style={{ textAlign:"center", color:GRAY, padding:40 }}>
                      {emps.length===0 ? "No employees yet — add your first one!" : "No results found."}
                    </td></tr>
                  )}
                </tbody>
              </table>
            )}
          </div>
        </div>

        {/* Profile panel */}
        {viewing && (
          <div className="card" style={{ width:280, flexShrink:0, borderTop:`2px solid ${GOLD}`, alignSelf:"flex-start", position:"sticky", top:0 }}>
            <div style={{ display:"flex", justifyContent:"space-between", alignItems:"flex-start", marginBottom:16 }}>
              <div style={{ fontFamily:"'Cinzel',serif", fontSize:13, color:GOLD, letterSpacing:1 }}>PROFILE</div>
              <button onClick={()=>setViewing(null)} style={{ background:"none", border:"none", color:GRAY, cursor:"pointer", fontSize:18 }}>✕</button>
            </div>
            <div style={{ textAlign:"center", marginBottom:16 }}>
              <Avatar name={viewing.name} size={60}/>
              <div style={{ fontFamily:"'Cinzel',serif", fontSize:16, color:WHITE, marginTop:10 }}>{viewing.name}</div>
              <div style={{ fontSize:13, color:GOLD, marginTop:4 }}>{viewing.designation||"—"}</div>
              <div style={{ marginTop:8 }}><StatusBadge s={viewing.status}/></div>
            </div>
            {[
              ["Department", viewing.department||"—"],
              ["Email", viewing.email||"—"],
              ["Phone", viewing.phone||"—"],
              ["CNIC", viewing.cnic||"—"],
              ["Gender", viewing.gender||"—"],
              ["Join Date", viewing.joinDate||"—"],
              ["Basic Salary", viewing.basicSalary ? `PKR ${Number(viewing.basicSalary).toLocaleString()}` : "—"],
              ["Punctuality Bonus", viewing.punctualityBonus ? `PKR ${Number(viewing.punctualityBonus).toLocaleString()}` : "PKR 0"],
              ["Gross Salary", viewing.grossSalary ? `PKR ${Number(viewing.grossSalary).toLocaleString()}` : "—"],
              ["PC Username", viewing.pcUsername||"—"],
              ["PC Password", viewing.pcPassword ? "••••••••" : "—"],
              ["Address", viewing.address||"—"],
              ["Father/Husband", viewing.fatherHusbandName||"—"],
            ["Date of Birth", viewing.dob||"—"],
            ["Marital Status", viewing.maritalStatus ? viewing.maritalStatus.charAt(0).toUpperCase()+viewing.maritalStatus.slice(1) : "—"],
            ["Gender", viewing.gender ? viewing.gender.charAt(0).toUpperCase()+viewing.gender.slice(1) : "—"],
            ["CNIC", viewing.cnic||"—"],
            ["Cell No", viewing.phone||"—"],
            ["Nationality", viewing.nationality||"—"],
            ["Religion", viewing.religion||"—"],
            ["Education", viewing.education||"—"],
            ["Address", viewing.address||"—"],
            ["Emergency Contact", `${viewing.emergencyContact||"—"}`],
            ["Emergency Person", viewing.emergencyName ? `${viewing.emergencyName} (${viewing.emergencyRelation||"—"})` : "—"],
            ["Joined Through", viewing.joinSource ? (viewing.joinSource==="referral" ? `Referral — ${viewing.referralName||"—"}` : viewing.joinSource.charAt(0).toUpperCase()+viewing.joinSource.slice(1)) : "—"],
            ["Basic Salary", viewing.basicSalary ? `PKR ${Number(viewing.basicSalary).toLocaleString()}` : "—"],
            ["Punctuality Bonus", `PKR ${Number(viewing.punctualityBonus||0).toLocaleString()}`],
            ["Gross Salary", `PKR ${(Number(viewing.basicSalary||0)+Number(viewing.punctualityBonus||0)).toLocaleString()}`],
            ].map(([k,v])=>(
              <div key={k} style={{ display:"flex", justifyContent:"space-between", padding:"8px 0", borderBottom:"1px solid #1a1a1a", gap:8 }}>
                <span style={{ fontSize:12, color:GRAY, flexShrink:0 }}>{k}</span>
                <span style={{ fontSize:12, color:WHITE, textAlign:"right", wordBreak:"break-all" }}>{v}</span>
              </div>
            ))}
            <button className="gold-btn" onClick={()=>openEdit(viewing)} style={{ width:"100%", marginTop:16, padding:"10px" }}>Edit Employee</button>
          </div>
        )}
      </div>

      {/* Add/Edit Modal */}
      {modal && (
        <div className="overlay" onClick={()=>setModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{ width:560, maxHeight:"90vh", overflowY:"auto" }}>
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:18, color:GOLD, marginBottom:4 }}>{editing?"Edit Employee":"New Employee"}</div>
            {!editing && <div style={{ fontSize:12, color:"#4CC98A", marginBottom:18, background:"rgba(76,201,138,.08)", border:"1px solid rgba(76,201,138,.2)", borderRadius:4, padding:"8px 12px" }}>
              🔐 Login credentials will be created automatically — Email: employee email, Password: CNIC (without dashes)
            </div>}
            {editing && <div style={{ fontSize:12, color:GRAY, marginBottom:18 }}>Edit employee details below</div>}
            <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:14 }}>
              {/* Personal Info */}
              {[
                ["Full Name","name","text","John Doe"],
                ["Father/Husband Name","fatherHusbandName","text","e.g. Muhammad Ali"],
                ["Email","email","email","john@falcoconnect.com"],
                ["Cell No","phone","text","+92 300 0000000"],
                ["CNIC","cnic","text","12345-1234567-1"],
                ["Date of Birth","dob","date",""],
                ["Nationality","nationality","text","Pakistani"],
                ["Religion","religion","text","Islam"],
                ["Highest Education","education","text","e.g. MBA, BBA, Matric"],
                ["Designation","designation","text","Manager"],
                ["Join Date","joinDate","date",""],
                ["Basic Salary (PKR)","basicSalary","number","50000"],
              ].map(([label,key,type,ph])=>(
                <div key={key}>
                  <label className="lbl">{label}</label>
                  <input style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
                    type={type} placeholder={ph} value={form[key]||""} onChange={e=>setForm({...form,[key]:e.target.value})}/>
                </div>
              ))}
              <div>
                <label className="lbl">Department</label>
                <select style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
                  value={form.department} onChange={e=>setForm({...form,department:e.target.value})}>
                  <option value="">— Select —</option>
                  {depts.map(d=><option key={d.id} value={d.name}>{d.name}</option>)}
                </select>
              </div>
              <div>
                <label className="lbl">Gender</label>
                <select style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
                  value={form.gender} onChange={e=>setForm({...form,gender:e.target.value})}>
                  <option value="male">Male</option>
                  <option value="female">Female</option>
                  <option value="other">Other</option>
                </select>
              </div>
              <div>
                <label className="lbl">Marital Status</label>
                <select style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
                  value={form.maritalStatus} onChange={e=>setForm({...form,maritalStatus:e.target.value})}>
                  <option value="single">Single</option>
                  <option value="married">Married</option>
                  <option value="divorced">Divorced</option>
                </select>
              </div>
              <div>
                <label className="lbl">Employment Status</label>
                <select style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
                  value={form.status} onChange={e=>setForm({...form,status:e.target.value})}>
                  <option value="active">Active</option>
                  <option value="on_leave">On Leave</option>
                  <option value="inactive">Inactive</option>
                </select>
              </div>

              {/* Emergency Contact */}
              <div style={{ gridColumn:"1/-1" }}>
                <div style={{ fontFamily:"'Cinzel',serif", fontSize:12, color:GOLD, letterSpacing:1, marginBottom:10 }}>🚨 EMERGENCY CONTACT</div>
                <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr 1fr", gap:12 }}>
                  <div>
                    <label className="lbl">Contact No</label>
                    <input value={form.emergencyContact||""} onChange={e=>setForm({...form,emergencyContact:e.target.value})}
                      placeholder="+92 300 0000000"
                      style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                  </div>
                  <div>
                    <label className="lbl">Contact Name</label>
                    <input value={form.emergencyName||""} onChange={e=>setForm({...form,emergencyName:e.target.value})}
                      placeholder="e.g. Ali Khan"
                      style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                  </div>
                  <div>
                    <label className="lbl">Relation</label>
                    <input value={form.emergencyRelation||""} onChange={e=>setForm({...form,emergencyRelation:e.target.value})}
                      placeholder="e.g. Brother, Father"
                      style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                  </div>
                </div>
              </div>

              {/* How employee joined */}
              <div>
                <label className="lbl">Joined Through</label>
                <select style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
                  value={form.joinSource} onChange={e=>setForm({...form,joinSource:e.target.value,referralName:""})}>
                  <option value="walk-in">Walk-in</option>
                  <option value="referral">Referral</option>
                  <option value="jobfair">Job Fair</option>
                  <option value="website">Website</option>
                  <option value="brochures">Brochures</option>
                  <option value="ads">Ads</option>
                </select>
              </div>
              {form.joinSource==="referral" && (
                <div>
                  <label className="lbl">Referral Name</label>
                  <input value={form.referralName||""} onChange={e=>setForm({...form,referralName:e.target.value})}
                    placeholder="Who referred this employee?"
                    style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                </div>
              )}
              <div style={{ gridColumn:"1/-1" }}>
                <label className="lbl">Address</label>
                <input style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
                  placeholder="Street, City, Province" value={form.address||""} onChange={e=>setForm({...form,address:e.target.value})}/>
              </div>

              {/* Salary Section */}
              <div style={{ gridColumn:"1/-1", borderTop:"1px solid #222", paddingTop:14, marginTop:4 }}>
                <div style={{ fontFamily:"'Cinzel',serif", fontSize:12, color:GOLD, letterSpacing:1, marginBottom:12 }}>💰 SALARY DETAILS</div>
                <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr 1fr", gap:12 }}>
                  <div>
                    <label className="lbl">Punctuality Bonus (PKR)</label>
                    <input type="number" placeholder="0" value={form.punctualityBonus}
                      onChange={e=>{
                        const bonus = e.target.value;
                        const gross = Number(form.basicSalary||0) + Number(bonus||0);
                        setForm({...form, punctualityBonus:bonus, grossSalary:gross});
                      }}
                      style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                  </div>
                  <div>
                    <label className="lbl">Gross Salary (Auto)</label>
                    <div style={{ background:"rgba(201,168,76,.08)", border:"1px solid rgba(201,168,76,.3)", borderRadius:4, padding:"11px 14px", fontFamily:"'Cinzel',serif", fontSize:15, color:GOLD, fontWeight:700 }}>
                      PKR {(Number(form.basicSalary||0) + Number(form.punctualityBonus||0)).toLocaleString()}
                    </div>
                  </div>
                </div>
              </div>

              {/* PC Credentials Section */}
              <div style={{ gridColumn:"1/-1", borderTop:"1px solid #222", paddingTop:14, marginTop:4 }}>
                <div style={{ fontFamily:"'Cinzel',serif", fontSize:12, color:GOLD, letterSpacing:1, marginBottom:12 }}>🖥️ PC CREDENTIALS</div>
                <div style={{ background:"rgba(201,76,76,.05)", border:"1px solid rgba(201,76,76,.15)", borderRadius:6, padding:"10px 14px", fontSize:12, color:GRAY, marginBottom:12 }}>
                  🔒 Stored securely — only visible to Super Admin & CEO
                </div>
                <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:12 }}>
                  <div>
                    <label className="lbl">PC Username</label>
                    <input value={form.pcUsername||""} onChange={e=>setForm({...form,pcUsername:e.target.value})}
                      placeholder="e.g. DREVANOX\kashif"
                      style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                  </div>
                  <div>
                    <label className="lbl">PC Password</label>
                    <input value={form.pcPassword||""} onChange={e=>setForm({...form,pcPassword:e.target.value})}
                      placeholder="PC login password"
                      style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                  </div>
                </div>
              </div>
            </div>
            {err && <div style={{ background:"rgba(201,76,76,.1)", border:"1px solid rgba(201,76,76,.3)", borderRadius:4, padding:"10px 14px", fontSize:13, color:"#C94C4C", marginTop:12 }}>⚠ {err}</div>}
            <div style={{ display:"flex", gap:12, marginTop:18 }}>
              <button className="gold-btn" onClick={save} disabled={saving} style={{ flex:1 }}>{saving?<Spinner size={14}/>:(editing?"Save Changes":"Add Employee")}</button>
              <button className="ghost-btn" onClick={()=>setModal(false)} style={{ flex:1 }}>Cancel</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}


// ─── ATTENDANCE MODULE ────────────────────────────────────────────────────────
function AttendancePage({ user }) {
  const [records,     setRecords]     = useState([]);
  const [employees,   setEmployees]   = useState([]);
  const [leaves,      setLeaves]      = useState([]);
  const [fineSettings,setFineSettings]= useState({ lateFine:1000, nncsFine:2000, lateGraceMinutes:10 });
  const [loading,     setLoading]     = useState(true);
  const [todayRec,    setTodayRec]    = useState(null);
  const [checking,    setChecking]    = useState(false);
  const [filterEmp,   setFilterEmp]   = useState("");
  const [filterMonth, setFilterMonth] = useState(new Date().toISOString().slice(0,7));
  const [viewMode,    setViewMode]    = useState("daily");
  const [selectedEmp, setSelectedEmp] = useState(null);
  const [manualModal, setManualModal] = useState(false);
  const [fineModal,   setFineModal]   = useState(false);
  const [manualForm,  setManualForm]  = useState({ employeeEmail:"", date:"", checkIn:"", checkOut:"", status:"present", notes:"" });
  const [manualSaving,setManualSaving]= useState(false);

  const isAdmin = ["super_admin","ceo","manager","hr_manager"].includes(user.role);
  const todayStr = new Date().toISOString().slice(0,10);

  // Weekend check
  const isWeekend = (dateStr) => { const d = new Date(dateStr+"T00:00:00").getDay(); return d===0||d===6; };
  const todayIsWeekend = isWeekend(todayStr);

  // Get PKT time
  const getPKTTime = () => {
    const now = new Date();
    const utc = now.getTime() + now.getTimezoneOffset()*60000;
    return new Date(utc + 5*60*60000);
  };

  // Parse time to decimal hours
  const parseHour = (t) => {
    if(!t) return null; t=t.trim();
    const isPM=t.toLowerCase().includes("pm"), isAM=t.toLowerCase().includes("am");
    const part=t.replace(/am|pm/gi,"").trim();
    let [h,m]=part.split(":").map(Number);
    if(isNaN(h)) return null; if(isNaN(m)) m=0;
    if(isPM&&h!==12) h+=12; if(isAM&&h===12) h=0;
    return h+m/60;
  };

  // Calculate hours between two times (overnight aware)
  const calcHours = (inTime, outTime) => {
    if(!inTime||!outTime) return null;
    const parse=t=>{ if(!t) return 0; t=t.trim();
      const isPM=t.toLowerCase().includes("pm"),isAM=t.toLowerCase().includes("am"),hasPer=isPM||isAM;
      const part=hasPer?t.replace(/am|pm/gi,"").trim():t;
      let [h,m]=part.split(":").map(Number); if(isNaN(h))h=0;if(isNaN(m))m=0;
      if(hasPer){if(isPM&&h!==12)h+=12;if(isAM&&h===12)h=0;} return h*60+m; };
    let inM=parse(inTime),outM=parse(outTime);
    if(outM<inM) outM+=24*60;
    const diff=outM-inM; if(diff<=0) return null;
    return {total:diff,display:`${Math.floor(diff/60)}h ${diff%60}m`,hrs:Math.floor(diff/60),mins:diff%60};
  };

  // Determine status from checkin/checkout
  const determineStatus = (checkIn, checkOut) => {
    const SHIFT_START=18, LATE_THRESH=19, EARLY_OUT=2;
    const graceMins = fineSettings.lateGraceMinutes||10;
    const inHour=parseHour(checkIn), outHour=parseHour(checkOut);
    let status="present",isLate=false,isHalfDay=false,fine=0,fineReason="";

    // Late: more than grace period after shift start (6PM)
    if(inHour!==null && inHour>(SHIFT_START+graceMins/60)){
      isLate=true;
    }
    // Half day: login after 7PM
    if(inHour!==null && inHour>=LATE_THRESH){ isHalfDay=true; status="half_day"; }
    // Fine only if late AND NOT half day
    if(isLate&&!isHalfDay){
      fine=fineSettings.lateFine||1000;
      fineReason=`Late coming (>${graceMins}min) — checked in at ${checkIn}`;
    }
    // Early out: before 2AM
    if(checkOut&&outHour!==null){
      const adjOut=outHour<14?outHour+24:outHour;
      if(adjOut<(EARLY_OUT+24)){ isHalfDay=true; status="half_day"; }
    }
    return {status,isLate,isHalfDay,fine,fineReason};
  };

  // Get NCNS count for employee
  const getNcnsCount = (email) => records.filter(r=>r.employeeEmail===email&&r.status==="ncns").length;

  // Get NCNS warning level
  const getNcnsWarning = (email) => {
    const count = getNcnsCount(email);
    if(count>=3) return {level:"terminated",label:"TERMINATED",color:"#C94C4C"};
    if(count===2) return {level:"warning2",label:"2nd NCNS — Next = Termination",color:"#C94C4C"};
    if(count===1) return {level:"warning1",label:"1st NCNS",color:"#C9A84C"};
    return null;
  };

  // Check if employee had approved leave on a date
  const hadApprovedLeave = (email, date) => leaves.some(l=>l.employeeEmail===email&&l.status==="approved"&&l.fromDate<=date&&l.toDate>=date);
  const hadRejectedLeave = (email, date) => leaves.some(l=>l.employeeEmail===email&&l.status==="rejected"&&l.fromDate<=date&&l.toDate>=date);

  useEffect(()=>{
    const u1=onSnapshot(collection(db,"attendance"),snap=>{setRecords(snap.docs.map(d=>({...d.data(),id:d.id}))); setLoading(false);});
    const u2=onSnapshot(collection(db,"employees"),snap=>setEmployees(snap.docs.map(d=>({...d.data(),id:d.id}))));
    const u3=onSnapshot(collection(db,"leaves"),snap=>setLeaves(snap.docs.map(d=>({...d.data(),id:d.id}))));
    const u4=onSnapshot(doc(db,"settings","fines"),snap=>{if(snap.exists())setFineSettings(snap.data());});
    const id=`${user.email}_${todayStr}`;
    const u5=onSnapshot(doc(db,"attendance",id),snap=>setTodayRec(snap.exists()?snap.data():null));
    return ()=>{u1();u2();u3();u4();u5();};
  },[user.email]);

  // Check In
  const checkIn = async () => {
    setChecking(true);
    const pkt=getPKTTime();
    const timeStr=pkt.toLocaleTimeString("en-PK",{hour:"2-digit",minute:"2-digit",hour12:true});
    const {isLate,fine,fineReason}=determineStatus(timeStr,null);
    await setDoc(doc(db,"attendance",`${user.email}_${todayStr}`),{
      employeeEmail:user.email,employeeName:user.name,date:todayStr,
      checkIn:timeStr,checkOut:null,status:"present",
      isLate,fine,fineReason,hoursWorked:null,minutesWorked:null,
      totalMinutes:null,displayHours:null,createdAt:serverTimestamp(),
    });
    setChecking(false);
  };

  // Check Out
  const checkOut = async () => {
    setChecking(true);
    const pkt=getPKTTime();
    const timeStr=pkt.toLocaleTimeString("en-PK",{hour:"2-digit",minute:"2-digit",hour12:true});
    const result=calcHours(todayRec.checkIn,timeStr);
    const {status,isHalfDay,fine,fineReason}=determineStatus(todayRec.checkIn,timeStr);
    const totalFine=(todayRec.fine||0)+fine;
    const allReasons=[todayRec.fineReason,fineReason].filter(Boolean).join("; ");
    await updateDoc(doc(db,"attendance",`${user.email}_${todayStr}`),{
      checkOut:timeStr,hoursWorked:result?result.hrs:0,minutesWorked:result?result.mins:0,
      totalMinutes:result?result.total:0,displayHours:result?result.display:"0h 0m",
      status,isHalfDay,fine:totalFine,fineReason:allReasons,
    });
    setChecking(false);
  };

  // Mark NCNS manually (admin)
  const markNcns = async (email, date, empName) => {
    if(!window.confirm(`Mark ${empName} as NCNS on ${date}?`)) return;
    const id=`${email}_${date}`;
    const ncnsCount = getNcnsCount(email)+1;
    const fine=fineSettings.nncsFine||2000;
    let warningNote="";
    if(ncnsCount===1) warningNote="1st NCNS — Fine applied";
    if(ncnsCount===2) warningNote="2nd NCNS — Written Warning issued";
    if(ncnsCount>=3) warningNote="3rd NCNS — TERMINATION";

    await setDoc(doc(db,"attendance",id),{
      employeeEmail:email,employeeName:empName,date,
      checkIn:null,checkOut:null,status:"ncns",
      fine,fineReason:`NCNS (No Call No Show) — ${warningNote}`,
      ncnsCount,warningNote,
      createdAt:serverTimestamp(),
      markedBy:user.email,
    },{merge:true});

    // Notification to managers
    await setDoc(doc(db,"notifications",`ncns_${email}_${date}`),{
      type:"ncns",employeeEmail:email,employeeName:empName,date,
      ncnsCount,warningNote,fine,
      message:`⚠ NCNS Alert: ${empName} — ${warningNote} on ${date}`,
      createdAt:serverTimestamp(),read:false,
    });
    alert(`✅ NCNS marked.\n${warningNote}\nFine: PKR ${fine.toLocaleString()}`);
  };

  // Manual attendance entry
  const saveManual = async () => {
    if(!manualForm.employeeEmail||!manualForm.date) return;
    setManualSaving(true);
    const id=`${manualForm.employeeEmail}_${manualForm.date}`;
    const result=manualForm.checkIn&&manualForm.checkOut?calcHours(manualForm.checkIn,manualForm.checkOut):null;
    const {status,isLate,isHalfDay,fine,fineReason}=manualForm.checkIn?determineStatus(manualForm.checkIn,manualForm.checkOut):{status:manualForm.status,isLate:false,isHalfDay:false,fine:0,fineReason:""};
    const empName=employees.find(e=>(e.email||e.id)===manualForm.employeeEmail)?.name||manualForm.employeeEmail;
    await setDoc(doc(db,"attendance",id),{
      employeeEmail:manualForm.employeeEmail,employeeName:empName,date:manualForm.date,
      checkIn:manualForm.checkIn||null,checkOut:manualForm.checkOut||null,
      status:manualForm.status,isLate,isHalfDay,fine,fineReason,
      hoursWorked:result?result.hrs:0,minutesWorked:result?result.mins:0,
      totalMinutes:result?result.total:0,displayHours:result?result.display:"—",
      notes:manualForm.notes||"Manual entry",manualEntry:true,
      addedBy:user.email,createdAt:serverTimestamp(),
    });
    setManualSaving(false); setManualModal(false);
    setManualForm({employeeEmail:"",date:"",checkIn:"",checkOut:"",status:"present",notes:""});
  };

  // Save fine settings
  const saveFineSettings = async () => {
    await setDoc(doc(db,"settings","fines"),fineSettings);
    setFineModal(false);
    alert("✅ Fine settings saved!");
  };

  // Filters
  const getWeekDates=()=>{
    const now=new Date(),day=now.getDay(),start=new Date(now);
    start.setDate(now.getDate()-day);
    return Array.from({length:7},(_,i)=>{const d=new Date(start);d.setDate(start.getDate()+i);return d.toISOString().slice(0,10);});
  };
  const filterRecs=(recs)=>{
    if(viewMode==="daily") return recs.filter(r=>r.date===todayStr);
    if(viewMode==="weekly") return recs.filter(r=>getWeekDates().includes(r.date));
    return recs.filter(r=>r.date?.startsWith(filterMonth));
  };
  const allFiltered=records.filter(r=>{
    const matchEmp=isAdmin?(r.employeeName||"").toLowerCase().includes(filterEmp.toLowerCase()):r.employeeEmail===user.email;
    const matchPeriod=viewMode==="daily"?r.date===todayStr:viewMode==="weekly"?getWeekDates().includes(r.date):r.date?.startsWith(filterMonth);
    return matchEmp&&matchPeriod;
  }).sort((a,b)=>b.date?.localeCompare(a.date));

  // Stats: if employee selected → show that employee, else show ALL
  const statRecs = selectedEmp
    ? filterRecs(records.filter(r=>r.employeeEmail===selectedEmp))
    : filterRecs(records); // ALL employees when none selected
  const presentDays = statRecs.filter(r=>r.status==="present").length;
  const halfDays    = statRecs.filter(r=>r.status==="half_day").length;
  const ncnsDays    = statRecs.filter(r=>r.status==="ncns").length;
  const absentDays  = statRecs.filter(r=>r.status==="absent").length;
  const totalMins   = statRecs.reduce((a,r)=>a+Number(r.totalMinutes||0),0);
  const totalFines  = statRecs.reduce((a,r)=>a+Number(r.fine||0),0);
  const lateDays    = statRecs.filter(r=>r.isLate&&!r.isHalfDay).length;
  const fmtMins=m=>{const n=Number(m)||0;return `${Math.floor(n/60)}h ${n%60}m`;};

  const StatusBadge=({r})=>{
    if(r.status==="ncns") return <span className="badge b-red">🚫 NCNS</span>;
    if(r.status==="half_day") return <span className="badge b-gold">½ Half Day</span>;
    if(r.status==="absent") return <span className="badge b-red">Absent</span>;
    return <span className="badge b-green">Present</span>;
  };

  // Notifications for admin
  const [notifications,setNotifications]=useState([]);
  useEffect(()=>{
    if(!isAdmin) return;
    const unsub=onSnapshot(collection(db,"notifications"),snap=>{
      setNotifications(snap.docs.map(d=>({...d.data(),id:d.id})).filter(n=>!n.read).sort((a,b)=>b.createdAt?.seconds-a.createdAt?.seconds));
    });
    return unsub;
  },[isAdmin]);

  const markNotifRead=async(id)=>{ await updateDoc(doc(db,"notifications",id),{read:true}); };

  return (
    <div className="fade">
      {/* NCNS Notifications for Admin */}
      {isAdmin && notifications.filter(n=>n.type==="ncns").length>0 && (
        <div style={{marginBottom:16}}>
          {notifications.filter(n=>n.type==="ncns").slice(0,3).map(n=>(
            <div key={n.id} style={{background:"rgba(201,76,76,.12)",border:"1px solid rgba(201,76,76,.4)",borderRadius:6,padding:"10px 16px",marginBottom:8,display:"flex",justifyContent:"space-between",alignItems:"center"}}>
              <span style={{fontSize:13,color:"#FF6F6F"}}>🚨 {n.message}</span>
              <button onClick={()=>markNotifRead(n.id)} style={{background:"none",border:"none",color:GRAY,cursor:"pointer",fontSize:12}}>✓ Dismiss</button>
            </div>
          ))}
        </div>
      )}

      {/* Header */}
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start",marginBottom:16}}>
        <div>
          <div style={{fontFamily:"'Cinzel',serif",fontSize:22,color:WHITE}}>Attendance</div>
          <div style={{color:GRAY,fontSize:12,marginTop:3}}>
            {todayIsWeekend&&<span style={{color:"#4C9AC9",fontWeight:600}}>🏖️ Weekend (Off) &nbsp;·&nbsp;</span>}
            Shift: 6PM–4AM PKT &nbsp;·&nbsp;
            <span style={{color:"#C94C4C"}}>Late {'>'}{fineSettings.lateGraceMinutes||10}min = PKR {(fineSettings.lateFine||1000).toLocaleString()} fine</span> &nbsp;·&nbsp;
            <span style={{color:"#C9A84C"}}>After 7PM or before 2AM = Half Day (no fine)</span>
          </div>
        </div>
        <div style={{display:"flex",gap:8,flexWrap:"wrap",justifyContent:"flex-end"}}>
          {isAdmin&&<button className="ghost-btn" onClick={()=>setFineModal(true)} style={{fontSize:12}}>⚙️ Fine Settings</button>}
          {isAdmin&&<button className="ghost-btn" onClick={()=>setManualModal(true)} style={{fontSize:12}}>✍️ Manual Entry</button>}
          {/* Check In/Out */}
          {todayIsWeekend?(
            <div style={{background:"rgba(76,154,201,.08)",border:"1px solid rgba(76,154,201,.2)",borderRadius:6,padding:"9px 16px",fontSize:13,color:"#4C9AC9",fontWeight:600}}>🏖️ Weekend Off</div>
          ):!todayRec?(
            <button onClick={checkIn} disabled={checking} style={{background:"rgba(76,201,138,.15)",border:"1px solid rgba(76,201,138,.5)",color:"#4CC98A",padding:"9px 20px",borderRadius:6,cursor:"pointer",fontFamily:"'Rajdhani',sans-serif",fontSize:14,fontWeight:700}}>
              {checking?<Spinner size={14}/>:"✅ Check In"}
            </button>
          ):!todayRec.checkOut?(
            <div style={{display:"flex",flexDirection:"column",alignItems:"flex-end",gap:4}}>
              <div style={{display:"flex",gap:8,alignItems:"center"}}>
                <span style={{fontSize:12,color:"#4CC98A"}}>✅ {todayRec.checkIn}</span>
                {todayRec.isLate&&!todayRec.isHalfDay&&<span style={{fontSize:11,color:"#C94C4C",fontWeight:700}}>⚠ LATE — PKR {(fineSettings.lateFine||1000).toLocaleString()}</span>}
              </div>
              <button onClick={checkOut} disabled={checking} style={{background:"rgba(201,76,76,.15)",border:"1px solid rgba(201,76,76,.5)",color:"#C94C4C",padding:"9px 20px",borderRadius:6,cursor:"pointer",fontFamily:"'Rajdhani',sans-serif",fontSize:14,fontWeight:700}}>
                {checking?<Spinner size={14}/>:"🚪 Check Out"}
              </button>
            </div>
          ):(
            <div style={{background:"rgba(201,168,76,.08)",border:"1px solid rgba(201,168,76,.2)",borderRadius:6,padding:"10px 16px",textAlign:"right"}}>
              <div style={{fontSize:13,color:"#4CC98A"}}>✅ {todayRec.checkIn} → {todayRec.checkOut}</div>
              <div style={{fontSize:16,color:GOLD,fontWeight:700}}>⏱ {todayRec.displayHours||"—"}</div>
              {todayRec.status==="half_day"&&<div style={{fontSize:11,color:"#C9A84C"}}>½ Half Day</div>}
              {todayRec.fine>0&&<div style={{fontSize:11,color:"#C94C4C"}}>Fine: PKR {Number(todayRec.fine).toLocaleString()}</div>}
            </div>
          )}
        </div>
      </div>

      {/* View Mode Tabs */}
      <div style={{display:"flex",gap:4,marginBottom:14,background:BLACK3,borderRadius:6,padding:4,width:"fit-content"}}>
        {[["daily","📅 Today"],["weekly","📆 Week"],["monthly","🗓 Monthly"]].map(([mode,lbl])=>(
          <button key={mode} onClick={()=>setViewMode(mode)} style={{background:viewMode===mode?GOLD:"transparent",color:viewMode===mode?BLACK:GRAY,border:"none",padding:"7px 14px",borderRadius:4,cursor:"pointer",fontFamily:"'Rajdhani',sans-serif",fontSize:13,fontWeight:700,transition:"all .2s"}}>{lbl}</button>
        ))}
      </div>
      {viewMode==="monthly"&&<div style={{marginBottom:12}}><input type="month" value={filterMonth} onChange={e=>setFilterMonth(e.target.value)} style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"8px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,outline:"none"}}/></div>}

      {/* Stats */}
      <div style={{display:"flex",gap:12,flexWrap:"wrap",marginBottom:16}}>
        {[
          {label:"Present",    val:presentDays,                          c:"#4CC98A", icon:"✅"},
          {label:"Absent",     val:absentDays,                           c:"#C94C4C", icon:"❌"},
          {label:"Half Days",  val:halfDays,                             c:"#C9A84C", icon:"½"},
          {label:"NCNS",       val:ncnsDays,                             c:"#C94C4C", icon:"🚫"},
          {label:"Late Coming",val:lateDays,                             c:"#C9A84C", icon:"⏰"},
          {label:"Total Hours",val:fmtMins(totalMins),                   c:GOLD,      icon:"⏱"},
          {label:"Total Fines",val:`PKR ${totalFines.toLocaleString()}`, c:"#C94C4C", icon:"⚠", sm:true},
        ].map(s=>(
          <div key={s.label} className="card" style={{flex:1,minWidth:110}}>
            <div style={{fontSize:11,color:GRAY,letterSpacing:1,textTransform:"uppercase",marginBottom:6}}>{s.icon} {s.label}</div>
            <div style={{fontSize:s.sm?13:22,fontFamily:"'Cinzel',serif",color:s.c,fontWeight:700}}>{s.val}</div>
          </div>
        ))}
      </div>

      <div style={{display:"flex",gap:20}}>
        {/* Employee list */}
        {isAdmin&&(
          <div style={{width:220,flexShrink:0}}>
            <div style={{marginBottom:8,position:"relative"}}>
              <span style={{position:"absolute",left:10,top:"50%",transform:"translateY(-50%)",color:GRAY,pointerEvents:"none",fontSize:13}}>⌕</span>
              <input style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"8px 10px 8px 30px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}
                placeholder="Search..." value={filterEmp} onChange={e=>{setFilterEmp(e.target.value);setSelectedEmp(null);}}/>
            </div>
            <div style={{background:BLACK2,border:"1px solid #222",borderRadius:6,overflow:"hidden",maxHeight:500,overflowY:"auto"}}>
              {employees.filter(e=>(e.name||"").toLowerCase().includes(filterEmp.toLowerCase())).map(e=>{
                const email=e.email||e.id;
                const empRecs=filterRecs(records.filter(r=>r.employeeEmail===email));
                const ep=empRecs.filter(r=>r.status==="present"||r.status==="half_day").length;
                const em=empRecs.reduce((a,r)=>a+Number(r.totalMinutes||0),0);
                const ef=empRecs.reduce((a,r)=>a+Number(r.fine||0),0);
                const ncnsWarn=getNcnsWarning(email);
                return (
                  <div key={e.id} onClick={()=>setSelectedEmp(email)} style={{padding:"10px 12px",borderBottom:"1px solid #1a1a1a",cursor:"pointer",background:selectedEmp===email?"rgba(201,168,76,.08)":"transparent",borderLeft:selectedEmp===email?`3px solid ${GOLD}`:"3px solid transparent",transition:"all .2s"}}>
                    <div style={{display:"flex",alignItems:"center",gap:8}}>
                      <Avatar name={e.name} size={26}/>
                      <div style={{flex:1,minWidth:0}}>
                        <div style={{fontSize:12,color:WHITE,fontWeight:500,whiteSpace:"nowrap",overflow:"hidden",textOverflow:"ellipsis"}}>{e.name}</div>
                        <div style={{fontSize:10,color:GRAY}}>{e.department||"—"}</div>
                      </div>
                    </div>
                    {ncnsWarn&&<div style={{fontSize:10,color:ncnsWarn.color,fontWeight:700,marginTop:4}}>⚠ {ncnsWarn.label}</div>}
                    <div style={{display:"flex",justifyContent:"space-between",marginTop:4}}>
                      <span style={{fontSize:10,color:"#4CC98A"}}>✅{ep}d</span>
                      <span style={{fontSize:10,color:GOLD}}>⏱{fmtMins(em)}</span>
                      {ef>0&&<span style={{fontSize:10,color:"#C94C4C"}}>⚠{ef.toLocaleString()}</span>}
                    </div>
                  </div>
                );
              })}
            </div>
          </div>
        )}

        {/* Table */}
        <div style={{flex:1}}>
          {/* NCNS Warning banner for selected employee */}
          {selectedEmp&&(()=>{
            const warn=getNcnsWarning(selectedEmp);
            const emp=employees.find(e=>(e.email||e.id)===selectedEmp);
            if(!warn) return null;
            return (
              <div style={{background:`${warn.color}18`,border:`1px solid ${warn.color}44`,borderRadius:6,padding:"10px 16px",marginBottom:12,display:"flex",justifyContent:"space-between",alignItems:"center"}}>
                <span style={{fontSize:13,color:warn.color,fontWeight:700}}>⚠ {emp?.name}: {warn.label}</span>
                {warn.level==="warning2"&&<span style={{fontSize:11,color:"#C94C4C"}}>Written Warning Issued</span>}
                {warn.level==="terminated"&&<span style={{fontSize:11,color:"#C94C4C",fontWeight:700}}>EMPLOYEE TERMINATED</span>}
              </div>
            );
          })()}

          <div className="card" style={{padding:0,overflow:"hidden"}}>
            {loading?<div style={{display:"flex",justifyContent:"center",padding:40}}><Spinner size={28}/></div>:(
              <table>
                <thead>
                  <tr>
                    {isAdmin&&!selectedEmp&&<th>Employee</th>}
                    <th>Date</th><th>Check In</th><th>Check Out</th><th>Duration</th><th>Status</th><th>Fine</th>
                    {isAdmin&&<th>Actions</th>}
                  </tr>
                </thead>
                <tbody>
                  {(selectedEmp?filterRecs(records.filter(r=>r.employeeEmail===selectedEmp)):allFiltered).map(r=>(
                    <tr key={r.id}>
                      {isAdmin&&!selectedEmp&&<td><div style={{display:"flex",alignItems:"center",gap:8}}><Avatar name={r.employeeName} size={26}/><span style={{fontSize:13}}>{r.employeeName}</span></div></td>}
                      <td style={{color:GOLD,fontSize:13,fontWeight:600}}>{r.date}</td>
                      <td><div style={{fontSize:13,color:r.isLate&&!r.isHalfDay?"#C94C4C":"#4CC98A",fontWeight:600}}>{r.checkIn||"—"}{r.isLate&&!r.isHalfDay&&<span style={{fontSize:10,marginLeft:4}}>⚠LATE</span>}</div></td>
                      <td style={{fontSize:13,color:"#C94C4C",fontWeight:600}}>{r.checkOut||<span style={{color:GRAY}}>—</span>}</td>
                      <td>{r.checkOut?(()=>{const res=calcHours(r.checkIn,r.checkOut);return <span style={{color:GOLD,fontWeight:700,fontSize:13}}>{res?res.display:r.displayHours||"—"}</span>;})():<span style={{color:GRAY,fontSize:12}}>In progress</span>}</td>
                      <td><StatusBadge r={r}/></td>
                      <td>{r.fine>0?<div><span style={{color:"#C94C4C",fontSize:12,fontWeight:700}}>PKR {Number(r.fine).toLocaleString()}</span><div style={{fontSize:10,color:GRAY}}>{r.fineReason}</div></div>:<span style={{color:GRAY,fontSize:12}}>—</span>}</td>
                      {isAdmin&&<td>
                        <div style={{display:"flex",gap:4}}>
                          {/* Only show NCNS button for absent employees or those who haven't checked in */}
                          {(r.status==="absent" || (!r.checkIn && r.status!=="ncns" && r.status!=="on_leave")) && (
                            <button onClick={()=>markNcns(r.employeeEmail,r.date,r.employeeName)} style={{background:"rgba(201,76,76,.1)",border:"1px solid rgba(201,76,76,.3)",color:"#C94C4C",padding:"3px 8px",borderRadius:4,cursor:"pointer",fontFamily:"'Rajdhani',sans-serif",fontSize:11,fontWeight:600}}>🚫 NCNS</button>
                          )}
                          {(r.status==="present"||r.status==="half_day") && (
                            <span style={{fontSize:11,color:"#444"}}>—</span>
                          )}
                          {r.status==="ncns" && (
                            <span style={{fontSize:11,color:"#C94C4C",fontWeight:700}}>NCNS ✓</span>
                          )}
                        </div>
                      </td>}
                    </tr>
                  ))}
                  {(selectedEmp?filterRecs(records.filter(r=>r.employeeEmail===selectedEmp)):allFiltered).length===0&&(
                    <tr><td colSpan={8} style={{textAlign:"center",color:GRAY,padding:36}}>No records for this period.</td></tr>
                  )}
                </tbody>
              </table>
            )}
          </div>
        </div>
      </div>

      {/* Manual Entry Modal */}
      {manualModal&&(
        <div className="overlay" onClick={()=>setManualModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{width:500}}>
            <div style={{fontFamily:"'Cinzel',serif",fontSize:18,color:GOLD,marginBottom:6}}>Manual Attendance Entry</div>
            <div style={{fontSize:12,color:"#4CC98A",background:"rgba(76,201,138,.08)",border:"1px solid rgba(76,201,138,.2)",borderRadius:4,padding:"8px 12px",marginBottom:16}}>
              Use for past dates or emergency absences
            </div>
            <div style={{display:"flex",flexDirection:"column",gap:14}}>
              <div>
                <label className="lbl">Employee</label>
                <select value={manualForm.employeeEmail} onChange={e=>setManualForm({...manualForm,employeeEmail:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}>
                  <option value="">— Select —</option>
                  {employees.map(e=><option key={e.id} value={e.email||e.id}>{e.name} — {e.department||""}</option>)}
                </select>
              </div>
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:12}}>
                <div><label className="lbl">Date</label><input type="date" value={manualForm.date} onChange={e=>setManualForm({...manualForm,date:e.target.value})} style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/></div>
                <div><label className="lbl">Status</label>
                  <select value={manualForm.status} onChange={e=>setManualForm({...manualForm,status:e.target.value})} style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}>
                    <option value="present">Present</option>
                    <option value="half_day">Half Day</option>
                    <option value="absent">Absent</option>
                    <option value="ncns">NCNS</option>
                  </select>
                </div>
                <div><label className="lbl">Check In (e.g. 06:00 PM)</label><input value={manualForm.checkIn} onChange={e=>setManualForm({...manualForm,checkIn:e.target.value})} placeholder="06:00 PM" style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/></div>
                <div><label className="lbl">Check Out (e.g. 04:00 AM)</label><input value={manualForm.checkOut} onChange={e=>setManualForm({...manualForm,checkOut:e.target.value})} placeholder="04:00 AM" style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/></div>
              </div>
              <div><label className="lbl">Notes</label><input value={manualForm.notes} onChange={e=>setManualForm({...manualForm,notes:e.target.value})} placeholder="Reason for manual entry" style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/></div>
              <div style={{display:"flex",gap:12}}>
                <button className="gold-btn" onClick={saveManual} disabled={manualSaving} style={{flex:1}}>{manualSaving?<Spinner size={14}/>:"Save Entry"}</button>
                <button className="ghost-btn" onClick={()=>setManualModal(false)} style={{flex:1}}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}

      {/* Fine Settings Modal */}
      {fineModal&&(
        <div className="overlay" onClick={()=>setFineModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{width:420}}>
            <div style={{fontFamily:"'Cinzel',serif",fontSize:18,color:GOLD,marginBottom:6}}>Fine Settings</div>
            <div style={{fontSize:12,color:GRAY,marginBottom:20}}>Adjustable by Manager, HR, CEO, Super Admin</div>
            <div style={{display:"flex",flexDirection:"column",gap:14}}>
              <div>
                <label className="lbl">Late Grace Period (minutes)</label>
                <input type="number" value={fineSettings.lateGraceMinutes||10} onChange={e=>setFineSettings({...fineSettings,lateGraceMinutes:Number(e.target.value)})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
                <div style={{fontSize:11,color:GRAY,marginTop:4}}>Fine applies only if late by more than this many minutes</div>
              </div>
              <div>
                <label className="lbl">Late Coming Fine (PKR)</label>
                <input type="number" value={fineSettings.lateFine||1000} onChange={e=>setFineSettings({...fineSettings,lateFine:Number(e.target.value)})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              <div>
                <label className="lbl">NCNS Fine (PKR)</label>
                <input type="number" value={fineSettings.nncsFine||2000} onChange={e=>setFineSettings({...fineSettings,nncsFine:Number(e.target.value)})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              <div style={{background:"rgba(201,168,76,.08)",border:"1px solid rgba(201,168,76,.2)",borderRadius:6,padding:"10px 14px",fontSize:12,color:GOLD}}>
                ℹ️ NCNS Rules:<br/>
                1st NCNS → Fine only<br/>
                2nd NCNS → Fine + Written Warning<br/>
                3rd NCNS → Termination
              </div>
              <div style={{display:"flex",gap:12}}>
                <button className="gold-btn" onClick={saveFineSettings} style={{flex:1}}>Save Settings</button>
                <button className="ghost-btn" onClick={()=>setFineModal(false)} style={{flex:1}}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}


// ─── PAYROLL MODULE ───────────────────────────────────────────────────────────
function PayrollPage({ user }) {
  const [payrolls,   setPayrolls]   = useState([]);
  const [employees,  setEmployees]  = useState([]);
  const [attendance, setAttendance] = useState([]);
  const [loading,    setLoading]    = useState(true);
  const [modal,      setModal]      = useState(false);
  const [editModal,  setEditModal]  = useState(false);
  const [settingsModal, setSettingsModal] = useState(false);
  const [viewing,    setViewing]    = useState(null);
  const [editing,    setEditing]    = useState(null);
  const [saving,     setSaving]     = useState(false);
  const [generating, setGenerating] = useState(false);
  const [filterMonth,setFilterMonth]= useState(new Date().toISOString().slice(0,7));
  const [payrollSettings, setPayrollSettings] = useState({ autoDay: 25, lastGenerated: "" });
  const [err, setErr] = useState("");
  const [notification, setNotification] = useState("");

  const EMPTY = { employeeEmail:"", employeeName:"", department:"", basicSalary:"", allowances:"0", deductions:"0", fines:"0", absentDays:"0", absentDeduction:"0", tax:"0", month: new Date().toISOString().slice(0,7), status:"pending", notes:"" };
  const [form, setForm] = useState(EMPTY);

  const isAdmin = ["super_admin","ceo","manager","hr_manager","finance_manager"].includes(user.role);

  useEffect(() => {
    const unsub1 = onSnapshot(collection(db,"payroll"), snap => {
      setPayrolls(snap.docs.map(d=>({...d.data(),id:d.id})));
      setLoading(false);
    });
    const unsub2 = onSnapshot(collection(db,"employees"), snap => {
      setEmployees(snap.docs.map(d=>({...d.data(),id:d.id})));
    });
    const unsub3 = onSnapshot(collection(db,"attendance"), snap => {
      setAttendance(snap.docs.map(d=>({...d.data(),id:d.id})));
    });
    // Load payroll settings
    const unsub4 = onSnapshot(doc(db,"settings","payroll"), snap => {
      if (snap.exists()) setPayrollSettings(snap.data());
    });
    return () => { unsub1(); unsub2(); unsub3(); unsub4(); };
  }, []);

  // Check if auto-payroll should run today
  useEffect(() => {
    if (!isAdmin) return;
    const today = new Date();
    const todayDate = today.getDate();
    const thisMonth = today.toISOString().slice(0,7);
    if (todayDate === payrollSettings.autoDay && payrollSettings.lastGenerated !== thisMonth) {
      setNotification(`📅 Today is payroll day (${payrollSettings.autoDay}th)! Click "Auto Generate" to process this month's payroll.`);
    }
  }, [payrollSettings]);

  // Calculate working days in a month (exclude Sat/Sun)
  const getWorkingDays = (month) => {
    const year = parseInt(month.split("-")[0]);
    const mon  = parseInt(month.split("-")[1]) - 1;
    const daysInMonth = new Date(year, mon+1, 0).getDate();
    let workingDays = 0;
    for(let d=1; d<=daysInMonth; d++){
      const day = new Date(year, mon, d).getDay();
      if(day !== 0 && day !== 6) workingDays++;
    }
    return { workingDays, daysInMonth };
  };

  // Calculate absent days for employee in a month — includes half-day = 0.5
  const getAbsentDays = (empEmail, month) => {
    const monthRecs = attendance.filter(r => r.employeeEmail===empEmail && r.date?.startsWith(month));
    const { workingDays } = getWorkingDays(month);
    let presentScore = 0;
    monthRecs.forEach(r=>{
      if(r.status==="present") presentScore += 1;
      else if(r.status==="half_day") presentScore += 0.5;
      // absent/ncns = 0
    });
    return Math.max(0, workingDays - presentScore);
  };

  // Sandwich rule: Friday or Monday unpaid leave/NCNS → +2 extra days deducted (weekend sandwiched)
  const getSandwichDeductionDays = (empEmail, month, leaves) => {
    const monthRecs = attendance.filter(r => r.employeeEmail===empEmail && r.date?.startsWith(month));
    let sandwichDays = 0;
    monthRecs.forEach(r=>{
      if(r.status!=="absent" && r.status!=="ncns") return;
      const dow = new Date(r.date+"T00:00:00").getDay();
      // Friday(5) or Monday(1) absence → sandwich the weekend (Sat+Sun = 2 extra days)
      if(dow===5 || dow===1) sandwichDays += 2;
    });
    // Also check unpaid leaves that fall on Friday or Monday
    const unpaidLeaves = leaves.filter(l=>
      l.employeeEmail===empEmail && l.leaveType==="unpaid" &&
      l.status==="approved" && l.fromDate?.startsWith(month)
    );
    unpaidLeaves.forEach(l=>{
      const dow = new Date(l.fromDate+"T00:00:00").getDay();
      if(dow===5 || dow===1) sandwichDays += 2;
    });
    return sandwichDays;
  };

  // Calculate per-day salary
  const calcPerDay = (basicSalary, month) => {
    const { daysInMonth } = getWorkingDays(month);
    return Number(basicSalary||0) / daysInMonth; // salary / total calendar days
  };

  // Auto generate payroll for all employees
  const autoGenerate = async () => {
    if (!window.confirm(`Generate payroll for ${filterMonth} for all ${employees.length} employees?`)) return;
    setGenerating(true);
    let count = 0;
    // Fetch all leaves once
    const leavesSnap = await getDocs(collection(db,"leaves"));
    const allLeaves = leavesSnap.docs.map(d=>d.data());
    for (const emp of employees) {
      const email = emp.email || emp.id;
      const id = `${email}_${filterMonth}`;
      // Check if already exists
      const existing = payrolls.find(p => p.id===id);
      if (existing) continue;
      const basic      = Number(emp.basicSalary||emp.salary||0);
      const punctBonus = Number(emp.punctualityBonus||0);
      const absentDays = getAbsentDays(email, filterMonth);
      const sandwichDays = getSandwichDeductionDays(email, filterMonth, allLeaves);
      const totalDeductDays = absentDays + sandwichDays;
      const perDay     = calcPerDay(basic, filterMonth);
      const absentDeduct = Math.round(totalDeductDays * perDay);
      // Unpaid leaves this month
      const unpaidCount  = allLeaves.filter(l=>
        l.employeeEmail===email && l.leaveType==="unpaid" &&
        l.status==="approved" && l.fromDate?.startsWith(filterMonth)
      ).length;
      // Deduct punctuality bonus if any unpaid leave
      const punctDeduct = unpaidCount > 0 ? punctBonus : 0;
      // Attendance fines (late coming)
      const attSnap  = await getDocs(collection(db,"attendance"));
      const attFines = attSnap.docs.map(d=>d.data()).filter(a=>
        a.employeeEmail===email && a.date?.startsWith(filterMonth) && a.fine>0
      ).reduce((x,a)=>x+Number(a.fine||0),0);
      const net = basic + punctBonus - absentDeduct - punctDeduct - attFines;
      await setDoc(doc(db,"payroll",id), {
        employeeEmail: email, employeeName: emp.name,
        department: emp.department||"", basicSalary: basic,
        allowances: 0, deductions: 0, fines: attFines,
        absentDays, sandwichDays, totalDeductDays, absentDeduction: absentDeduct,
        punctualityBonus: punctBonus,
        punctualityDeduction: punctDeduct,
        unpaidLeaves: unpaidCount,
        tax: 0, netSalary: net, month: filterMonth,
        status:"pending", notes:`Auto-generated. Absent: ${absentDays}d${sandwichDays>0?` + ${sandwichDays}d sandwich`:""}. ${unpaidCount>0?`Punctuality deducted (${unpaidCount} unpaid leave).`:""} ${attFines>0?`Late fines: PKR ${attFines}.`:""}`,
        generatedAt: serverTimestamp(),
      });
      count++;
    }
    // Update last generated
    await setDoc(doc(db,"settings","payroll"), { ...payrollSettings, lastGenerated: filterMonth });
    setGenerating(false);
    setNotification(`✅ Payroll generated for ${count} employees!`);
    setTimeout(()=>setNotification(""), 4000);
  };

  const displayed = payrolls.filter(p =>
    p.month === filterMonth &&
    (isAdmin ? true : p.employeeEmail === user.email)
  ).sort((a,b)=>(a.employeeName||"").localeCompare(b.employeeName||""));

  // Open edit modal
  const openEdit = (p) => {
    setEditing(p);
    setForm({
      ...p,
      allowances: p.allowances||0,
      deductions: p.deductions||0,
      fines: p.fines||0,
      absentDays: p.absentDays||0,
      absentDeduction: p.absentDeduction||0,
      tax: p.tax||0,
    });
    setEditModal(true);
  };

  // Recalculate net when form changes
  const getNet = (f) => {
    const basic  = Number(f.basicSalary)||0;
    const allow  = Number(f.allowances)||0;
    const deduct = Number(f.deductions)||0;
    const fines  = Number(f.fines)||0;
    const absent = Number(f.absentDeduction)||0;
    const tax    = Number(f.tax)||0;
    return basic + allow - deduct - fines - absent - tax;
  };

  const saveEdit = async () => {
    setSaving(true);
    const net = getNet(form);
    await updateDoc(doc(db,"payroll",editing.id), {
      ...form,
      allowances: Number(form.allowances)||0,
      deductions: Number(form.deductions)||0,
      fines: Number(form.fines)||0,
      absentDays: Number(form.absentDays)||0,
      absentDeduction: Number(form.absentDeduction)||0,
      tax: Number(form.tax)||0,
      netSalary: net,
      editedAt: serverTimestamp(),
      editedBy: user.email,
    });
    setSaving(false); setEditModal(false);
  };

  const markPaid = async (p) => {
    await updateDoc(doc(db,"payroll",p.id), { status:"paid", paidAt: serverTimestamp(), paidBy: user.email });
  };

  const del = async (p) => {
    if (!window.confirm(`Delete payroll for ${p.employeeName}?`)) return;
    await deleteDoc(doc(db,"payroll",p.id));
  };

  const saveSettings = async () => {
    await setDoc(doc(db,"settings","payroll"), payrollSettings);
    setSettingsModal(false);
  };

  const PKR = n => `PKR ${Number(n||0).toLocaleString()}`;
  const totalNet  = displayed.reduce((a,p)=>a+Number(p.netSalary||0),0);
  const totalPaid = displayed.filter(p=>p.status==="paid").length;
  const totalPend = displayed.filter(p=>p.status==="pending").length;

  return (
    <div className="fade">
      {/* Notification Banner */}
      {notification && (
        <div style={{ background:"rgba(201,168,76,.15)", border:"1px solid rgba(201,168,76,.4)", borderRadius:6, padding:"12px 18px", marginBottom:20, fontSize:14, color:GOLD, display:"flex", justifyContent:"space-between", alignItems:"center" }}>
          <span>{notification}</span>
          <button onClick={()=>setNotification("")} style={{ background:"none", border:"none", color:GOLD, cursor:"pointer", fontSize:18 }}>✕</button>
        </div>
      )}

      {/* Header */}
      <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:20 }}>
        <div>
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE }}>Payroll</div>
          <div style={{ color:GRAY, fontSize:14, marginTop:4 }}>
            Auto-generate on <span style={{color:GOLD}}>{payrollSettings.autoDay}th</span> of every month
          </div>
        </div>
        {isAdmin && (
          <div style={{ display:"flex", gap:10 }}>
            <button className="ghost-btn" onClick={()=>setSettingsModal(true)} style={{ fontSize:13 }}>⚙️ Payroll Settings</button>
            <button className="gold-btn" onClick={autoGenerate} disabled={generating}>
              {generating ? <><Spinner size={14}/> Generating...</> : "⚡ Auto Generate"}
            </button>
          </div>
        )}
      </div>

      {/* Stats */}
      {isAdmin && (
        <div style={{ display:"flex", gap:14, flexWrap:"wrap", marginBottom:20 }}>
          {[
            { label:"Total Payroll", val:PKR(totalNet), c:GOLD },
            { label:"Paid", val:totalPaid, c:"#4CC98A" },
            { label:"Pending", val:totalPend, c:"#C94C4C" },
            { label:"Employees", val:displayed.length, c:"#4C9AC9" },
          ].map(s=>(
            <div key={s.label} className="card" style={{ flex:1, minWidth:130 }}>
              <div style={{ fontSize:11, color:GRAY, letterSpacing:1, textTransform:"uppercase", marginBottom:8 }}>{s.label}</div>
              <div style={{ fontSize:s.label==="Total Payroll"?20:28, fontFamily:"'Cinzel',serif", color:s.c, fontWeight:700 }}>{s.val}</div>
            </div>
          ))}
        </div>
      )}

      {/* Month Filter */}
      <div style={{ display:"flex", gap:12, marginBottom:18, alignItems:"center" }}>
        <input type="month" value={filterMonth} onChange={e=>setFilterMonth(e.target.value)}
          style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"9px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, outline:"none" }}/>
        <span style={{ fontSize:13, color:GRAY }}>{displayed.length} records</span>
      </div>

      {/* Table */}
      <div className="card" style={{ padding:0, overflow:"hidden" }}>
        {loading ? (
          <div style={{ display:"flex", justifyContent:"center", padding:40 }}><Spinner size={28}/></div>
        ) : (
          <table>
            <thead>
              <tr>
                <th>Employee</th>
                <th>Basic</th>
                <th>Absent Days</th>
                <th>Absent Deduct</th>
                <th>Allowances</th>
                <th>Fines</th>
                <th>Tax</th>
                <th>Net Salary</th>
                <th>Status</th>
                {isAdmin && <th>Actions</th>}
              </tr>
            </thead>
            <tbody>
              {displayed.map(p=>(
                <tr key={p.id} style={{ cursor:"pointer" }} onClick={()=>setViewing(p)}>
                  <td>
                    <div style={{ display:"flex", alignItems:"center", gap:8 }}>
                      <Avatar name={p.employeeName} size={28}/>
                      <div>
                        <div style={{ fontSize:13, fontWeight:600 }}>{p.employeeName}</div>
                        <div style={{ fontSize:11, color:GRAY }}>{p.department}</div>
                      </div>
                    </div>
                  </td>
                  <td style={{ fontSize:13 }}>{PKR(p.basicSalary)}</td>
                  <td style={{ color:"#C94C4C", fontSize:13, fontWeight:600 }}>{p.absentDays||0} days</td>
                  <td style={{ color:"#C94C4C", fontSize:13 }}>-{PKR(p.absentDeduction)}</td>
                  <td style={{ color:"#4CC98A", fontSize:13 }}>+{PKR(p.allowances)}</td>
                  <td style={{ color:"#C94C9A", fontSize:13 }}>-{PKR(p.fines)}</td>
                  <td style={{ color:"#888", fontSize:13 }}>-{PKR(p.tax)}</td>
                  <td style={{ color:GOLD, fontWeight:700, fontSize:14 }}>{PKR(p.netSalary)}</td>
                  <td onClick={e=>e.stopPropagation()}>
                    <span className={`badge ${p.status==="paid"?"b-green":"b-gold"}`}>
                      {p.status==="paid"?"✅ Paid":"⏳ Pending"}
                    </span>
                  </td>
                  {isAdmin && (
                    <td onClick={e=>e.stopPropagation()}>
                      <div style={{ display:"flex", gap:6 }}>
                        <button className="ghost-btn" onClick={()=>openEdit(p)} style={{ padding:"4px 10px", fontSize:12 }}>✏️ Edit</button>
                        {p.status!=="paid" && (
                          <button className="ghost-btn" onClick={()=>markPaid(p)} style={{ padding:"4px 10px", fontSize:12, borderColor:"#4CC98A", color:"#4CC98A" }}>✅ Paid</button>
                        )}
                        <button className="danger-btn" onClick={()=>del(p)} style={{ padding:"4px 8px" }}>✕</button>
                      </div>
                    </td>
                  )}
                </tr>
              ))}
              {displayed.length===0&&!loading&&(
                <tr><td colSpan={isAdmin?10:9} style={{ textAlign:"center", color:GRAY, padding:40 }}>
                  No payroll for {filterMonth}. {isAdmin&&"Click 'Auto Generate' to create."}
                </td></tr>
              )}
            </tbody>
          </table>
        )}
      </div>

      {/* Payslip View Modal */}
      {viewing && (
        <div className="overlay" onClick={()=>setViewing(null)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{ width:440 }}>
            <div style={{ textAlign:"center", marginBottom:20 }}>
              <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:GOLD }}>FALCO CONNECT</div>
              <div style={{ fontSize:11, color:GRAY, letterSpacing:3 }}>SALARY SLIP</div>
            </div>
            <div style={{ borderTop:`1px solid #333`, padding:"14px 0", marginBottom:12 }}>
              {[["Employee",viewing.employeeName],["Department",viewing.department||"—"],["Month",viewing.month],["Notes",viewing.notes||"—"]].map(([k,v])=>(
                <div key={k} style={{ display:"flex", justifyContent:"space-between", padding:"6px 0" }}>
                  <span style={{ color:GRAY, fontSize:13 }}>{k}</span>
                  <span style={{ color:WHITE, fontSize:13 }}>{v}</span>
                </div>
              ))}
            </div>
            {[
              ["Basic Salary", PKR(viewing.basicSalary), WHITE],
              [`Absent (${viewing.absentDays||0} days)`, `-${PKR(viewing.absentDeduction)}`, "#C94C4C"],
              ["Allowances (+)", `+${PKR(viewing.allowances)}`, "#4CC98A"],
              ["Deductions (-)", `-${PKR(viewing.deductions)}`, "#C94C4C"],
              ["Fines (-)", `-${PKR(viewing.fines)}`, "#C94C9A"],
              ["Tax (-)", `-${PKR(viewing.tax)}`, "#888"],
            ].map(([k,v,c])=>(
              <div key={k} style={{ display:"flex", justifyContent:"space-between", padding:"9px 0", borderBottom:"1px solid #1a1a1a" }}>
                <span style={{ fontSize:13, color:GRAY }}>{k}</span>
                <span style={{ fontSize:13, color:c, fontWeight:500 }}>{v}</span>
              </div>
            ))}
            <div style={{ display:"flex", justifyContent:"space-between", padding:"14px 0", borderTop:`2px solid ${GOLD}`, marginTop:4 }}>
              <span style={{ fontFamily:"'Cinzel',serif", fontSize:15, color:WHITE }}>NET SALARY</span>
              <span style={{ fontFamily:"'Cinzel',serif", fontSize:20, color:GOLD, fontWeight:700 }}>{PKR(viewing.netSalary)}</span>
            </div>
            <div style={{ display:"flex", gap:10, marginTop:16 }}>
              {isAdmin && <button className="ghost-btn" onClick={()=>{setViewing(null);openEdit(viewing);}} style={{ flex:1 }}>✏️ Edit</button>}
              <button className="ghost-btn" onClick={()=>setViewing(null)} style={{ flex:1 }}>Close</button>
            </div>
          </div>
        </div>
      )}

      {/* Edit Payroll Modal */}
      {editModal && editing && (
        <div className="overlay" onClick={()=>setEditModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{ width:520 }}>
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:18, color:GOLD, marginBottom:6 }}>Edit Payroll</div>
            <div style={{ fontSize:13, color:GRAY, marginBottom:20 }}>{editing.employeeName} — {editing.month}</div>
            <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:14 }}>
              {[
                ["Basic Salary","basicSalary"],
                ["Allowances","allowances"],
                ["Other Deductions","deductions"],
                ["Fines","fines"],
                ["Absent Days","absentDays"],
                ["Absent Deduction","absentDeduction"],
                ["Tax","tax"],
              ].map(([label,key])=>(
                <div key={key}>
                  <label className="lbl">{label}</label>
                  <input type="number" value={form[key]} onChange={e=>{
                    const newForm = {...form,[key]:e.target.value};
                    // Auto calc absent deduction when absent days change
                    if(key==="absentDays") {
                      newForm.absentDeduction = Math.round(Number(e.target.value) * calcPerDay(form.basicSalary));
                    }
                    setForm(newForm);
                  }}
                    style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                </div>
              ))}
              <div>
                <label className="lbl">Status</label>
                <select value={form.status} onChange={e=>setForm({...form,status:e.target.value})}
                  style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}>
                  <option value="pending">Pending</option>
                  <option value="paid">Paid</option>
                </select>
              </div>
              <div style={{ gridColumn:"1/-1" }}>
                <label className="lbl">Notes</label>
                <input value={form.notes||""} onChange={e=>setForm({...form,notes:e.target.value})}
                  style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
              </div>
            </div>
            {/* Net Preview */}
            <div style={{ background:"rgba(201,168,76,.08)", border:"1px solid rgba(201,168,76,.2)", borderRadius:6, padding:"12px 16px", display:"flex", justifyContent:"space-between", alignItems:"center", marginTop:14 }}>
              <span style={{ fontSize:13, color:GRAY }}>Net Salary Preview</span>
              <span style={{ fontFamily:"'Cinzel',serif", fontSize:20, color:GOLD, fontWeight:700 }}>{PKR(getNet(form))}</span>
            </div>
            <div style={{ display:"flex", gap:12, marginTop:16 }}>
              <button className="gold-btn" onClick={saveEdit} disabled={saving} style={{ flex:1 }}>{saving?<Spinner size={14}/>:"Save Changes"}</button>
              <button className="ghost-btn" onClick={()=>setEditModal(false)} style={{ flex:1 }}>Cancel</button>
            </div>
          </div>
        </div>
      )}

      {/* Payroll Settings Modal */}
      {settingsModal && (
        <div className="overlay" onClick={()=>setSettingsModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{ width:400 }}>
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:18, color:GOLD, marginBottom:6 }}>Payroll Settings</div>
            <div style={{ fontSize:13, color:GRAY, marginBottom:20 }}>Configure auto-payroll generation</div>
            <div style={{ display:"flex", flexDirection:"column", gap:14 }}>
              <div>
                <label className="lbl">Auto-Generate on which day of month?</label>
                <input type="number" min="1" max="31" value={payrollSettings.autoDay}
                  onChange={e=>setPayrollSettings({...payrollSettings,autoDay:Number(e.target.value)})}
                  style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                <div style={{ fontSize:12, color:GRAY, marginTop:6 }}>Currently set to: <span style={{color:GOLD}}>{payrollSettings.autoDay}th of every month</span></div>
              </div>
              <div style={{ background:"rgba(201,168,76,.08)", border:"1px solid rgba(201,168,76,.2)", borderRadius:6, padding:"12px" }}>
                <div style={{ fontSize:13, color:GOLD, marginBottom:6 }}>ℹ️ How Auto-Payroll Works</div>
                <div style={{ fontSize:12, color:GRAY, lineHeight:1.6 }}>
                  • Opens app on the set date → notification appears<br/>
                  • Click "Auto Generate" → payroll for ALL employees created<br/>
                  • Basic Salary - Absent Day Deductions = Net Salary<br/>
                  • Edit individual records to add fines/allowances<br/>
                  • Mark as "Paid" when salary is disbursed
                </div>
              </div>
              <div style={{ display:"flex", gap:12 }}>
                <button className="gold-btn" onClick={saveSettings} style={{ flex:1 }}>Save Settings</button>
                <button className="ghost-btn" onClick={()=>setSettingsModal(false)} style={{ flex:1 }}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}


// ─── LEAVE MANAGEMENT MODULE ──────────────────────────────────────────────────
function LeavePage({ user }) {
  const [leaves,    setLeaves]    = useState([]);
  const [loading,   setLoading]   = useState(true);
  const [modal,     setModal]     = useState(false);
  const [saving,    setSaving]    = useState(false);
  const [filterStatus, setFilterStatus] = useState("all");
  const [filterMonth,  setFilterMonth]  = useState("");
  const [search,    setSearch]    = useState("");
  const [err,       setErr]       = useState("");

  const EMPTY = { leaveType:"annual", fromDate:"", toDate:"", reason:"", status:"pending" };
  const [form, setForm] = useState(EMPTY);

  const isAdmin = ["super_admin","ceo","manager","hr_manager"].includes(user.role);

  const LEAVE_TYPES = [
    { value:"monthly_paid", label:"Monthly Paid Leave", color:"#4CC98A", days:1, perMonth:true },
    { value:"annual",       label:"Annual Leave",       color:"#4C9AC9", days:12, requiresOneYear:true },
    { value:"unpaid",       label:"Unpaid Leave",       color:"#888888", days:999 },
  ];

  useEffect(() => {
    const unsub = onSnapshot(collection(db,"leaves"), snap => {
      setLeaves(snap.docs.map(d=>({...d.data(),id:d.id})));
      setLoading(false);
    });
    return unsub;
  }, []);

  // Calculate days between two dates
  const calcDays = (from, to) => {
    if (!from||!to) return 0;
    const d1 = new Date(from), d2 = new Date(to);
    const diff = Math.ceil((d2-d1)/(1000*60*60*24))+1;
    return diff > 0 ? diff : 0;
  };

  // Get leave balance for current user
  const getBalance = (type, empJoinDate) => {
    const lt = LEAVE_TYPES.find(l=>l.value===type);
    if (!lt || lt.days===999) return { label:"Unpaid", available:true };

    // Annual leave — check if 12 months completed
    if (lt.requiresOneYear) {
      if (!empJoinDate) return { label:"Join date required", available:false };
      const join = new Date(empJoinDate);
      const now  = new Date();
      const monthsDiff = (now.getFullYear()-join.getFullYear())*12 + (now.getMonth()-join.getMonth());
      if (monthsDiff < 12) {
        const remaining = 12 - monthsDiff;
        return { label:`Unlocks in ${remaining} month(s)`, available:false };
      }
      const used = leaves.filter(l=>
        l.employeeEmail===user.email && l.leaveType==="annual" &&
        l.status==="approved" && new Date(l.fromDate).getFullYear()===now.getFullYear()
      ).reduce((a,l)=>a+Number(l.days||0),0);
      return { label:`${lt.days-used} / ${lt.days} days`, available:lt.days-used>0 };
    }

    // Monthly paid leave — 1 per month, only after 3 months probation
    if (lt.perMonth) {
      // Check probation — need joinDate from employee record
      if (empJoinDate) {
        const join = new Date(empJoinDate);
        const now  = new Date();
        const monthsDiff = (now.getFullYear()-join.getFullYear())*12 + (now.getMonth()-join.getMonth());
        if (monthsDiff < 3) {
          const remaining = 3 - monthsDiff;
          return { label:`Probation — unlocks in ${remaining} month(s)`, available:false, probation:true };
        }
      }
      const thisMonth = new Date().toISOString().slice(0,7);
      const usedThisMonth = leaves.filter(l=>
        l.employeeEmail===user.email && l.leaveType==="monthly_paid" &&
        l.status==="approved" && l.fromDate?.startsWith(thisMonth)
      ).length;
      return { label: usedThisMonth>=1 ? "Used (0 remaining)" : "1 available", available: usedThisMonth<1 };
    }

    return { label:"Available", available:true };
  };

  // Validate leave before applying
  const validateLeave = (f, empJoinDate) => {
    if (f.leaveType==="annual") {
      const bal = getBalance("annual", empJoinDate);
      if (!bal.available) return `Annual leave not available: ${bal.label}`;
      const days = calcDays(f.fromDate, f.toDate);
      const used = leaves.filter(l=>l.employeeEmail===user.email&&l.leaveType==="annual"&&l.status==="approved"&&new Date(l.fromDate).getFullYear()===new Date().getFullYear()).reduce((a,l)=>a+Number(l.days||0),0);
      if (used + days > 12) return `Only ${12-used} annual leave days remaining.`;
    }
    if (f.leaveType==="monthly_paid") {
      const bal = getBalance("monthly_paid", empJoinDate);
      if (bal.probation) return `❌ Paid leave not available during probation period. ${bal.label}`;
      if (!bal.available) return "Monthly paid leave already used this month.";
    }
    return null;
  };

  const applyLeave = async () => {
    if (!form.fromDate||!form.toDate||!form.reason) { setErr("All fields required."); return; }
    if (new Date(form.toDate) < new Date(form.fromDate)) { setErr("End date must be after start date."); return; }
    // Validate leave rules
    const validErr = validateLeave(form, user.joinDate);
    if (validErr) { setErr(validErr); return; }
    setSaving(true); setErr("");
    const days = calcDays(form.fromDate, form.toDate);
    const id   = `${user.email}_${Date.now()}`;
    await setDoc(doc(db,"leaves",id), {
      ...form, id,
      employeeEmail: user.email,
      employeeName:  user.name,
      department:    user.department||"",
      days,
      status: "pending",
      appliedAt: serverTimestamp(),
    });
    setSaving(false); setModal(false); setForm(EMPTY);
  };

  const updateStatus = async (leave, status) => {
    await updateDoc(doc(db,"leaves",leave.id), {
      status,
      reviewedBy:   user.email,
      reviewedAt:   serverTimestamp(),
      reviewerName: user.name,
    });
  };

  const del = async (leave) => {
    if (!window.confirm("Delete this leave request?")) return;
    await deleteDoc(doc(db,"leaves",leave.id));
  };

  // Filter leaves
  const displayed = leaves.filter(l => {
    const matchUser   = isAdmin ? true : l.employeeEmail===user.email;
    const matchStatus = filterStatus==="all" ? true : l.status===filterStatus;
    const matchMonth  = filterMonth ? l.fromDate?.startsWith(filterMonth) : true;
    const matchSearch = isAdmin ? (l.employeeName||"").toLowerCase().includes(search.toLowerCase()) : true;
    return matchUser && matchStatus && matchMonth && matchSearch;
  }).sort((a,b) => b.appliedAt?.seconds - a.appliedAt?.seconds);

  const StatusBadge = ({s}) => {
    const cfg = {
      pending:  { cls:"b-gold",  label:"⏳ Pending"  },
      approved: { cls:"b-green", label:"✅ Approved" },
      rejected: { cls:"b-red",   label:"❌ Rejected" },
    };
    const c = cfg[s]||cfg.pending;
    return <span className={`badge ${c.cls}`}>{c.label}</span>;
  };

  const LeaveTypeBadge = ({t}) => {
    const lt = LEAVE_TYPES.find(l=>l.value===t);
    return <span style={{ fontSize:12, color:lt?.color||GOLD, background:`${lt?.color||GOLD}18`, border:`1px solid ${lt?.color||GOLD}44`, padding:"2px 10px", borderRadius:20, fontWeight:600 }}>{lt?.label||t}</span>;
  };

  return (
    <div className="fade">
      {/* Header */}
      <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:20 }}>
        <div>
          <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE }}>Leave Management</div>
          <div style={{ color:GRAY, fontSize:14, marginTop:4 }}>
            {isAdmin ? `${leaves.filter(l=>l.status==="pending").length} pending approvals` : "Apply and track your leaves"}
          </div>
        </div>
        <button className="gold-btn" onClick={()=>{setForm(EMPTY);setErr("");setModal(true);}}>+ Apply Leave</button>
      </div>

      {/* Leave Balance Cards (employee view) */}
      {!isAdmin && (
        <div style={{ display:"flex", gap:12, flexWrap:"wrap", marginBottom:20 }}>
          {LEAVE_TYPES.map(lt=>{
            const bal = lt.days===999 ? { label:"Unlimited (Unpaid)", available:true } : getBalance(lt.value, user.joinDate);
            return (
              <div key={lt.value} className="card" style={{ flex:1, minWidth:160, borderTop:`3px solid ${bal.available?lt.color:"#444"}` }}>
                <div style={{ fontSize:11, color:GRAY, letterSpacing:1, textTransform:"uppercase", marginBottom:8 }}>{lt.label}</div>
                <div style={{ fontSize:14, fontFamily:"'Cinzel',serif", color:bal.probation?"#C94C4C":bal.available?lt.color:"#555", fontWeight:700 }}>{bal.label}</div>
                {lt.requiresOneYear && <div style={{ fontSize:11, color:GRAY, marginTop:4 }}>Requires 12 months service</div>}
                {lt.perMonth && !bal.probation && <div style={{ fontSize:11, color:GRAY, marginTop:4 }}>1 paid leave per month</div>}
                {lt.perMonth && bal.probation && <div style={{ fontSize:11, color:"#C94C4C", marginTop:4 }}>⚠ 3 months probation required</div>}
              </div>
            );
          })}
        </div>
      )}

      {/* Admin Stats */}
      {isAdmin && (
        <div style={{ display:"flex", gap:14, flexWrap:"wrap", marginBottom:20 }}>
          {[
            { label:"Pending",  val:leaves.filter(l=>l.status==="pending").length,  c:"#C9A84C" },
            { label:"Approved", val:leaves.filter(l=>l.status==="approved").length, c:"#4CC98A" },
            { label:"Rejected", val:leaves.filter(l=>l.status==="rejected").length, c:"#C94C4C" },
            { label:"Total",    val:leaves.length, c:"#4C9AC9" },
          ].map(s=>(
            <div key={s.label} className="card" style={{ flex:1, minWidth:120 }}>
              <div style={{ fontSize:11, color:GRAY, letterSpacing:1, textTransform:"uppercase", marginBottom:8 }}>{s.label}</div>
              <div style={{ fontSize:28, fontFamily:"'Cinzel',serif", color:s.c, fontWeight:700 }}>{s.val}</div>
            </div>
          ))}
        </div>
      )}

      {/* Filters */}
      <div style={{ display:"flex", gap:12, marginBottom:18, flexWrap:"wrap" }}>
        {isAdmin && (
          <div style={{ position:"relative", flex:1, maxWidth:280 }}>
            <span style={{ position:"absolute", left:10, top:"50%", transform:"translateY(-50%)", color:GRAY, pointerEvents:"none" }}>⌕</span>
            <input style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"9px 10px 9px 30px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}
              placeholder="Search employee..." value={search} onChange={e=>setSearch(e.target.value)}/>
          </div>
        )}
        <select value={filterStatus} onChange={e=>setFilterStatus(e.target.value)}
          style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"9px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, outline:"none", cursor:"pointer" }}>
          <option value="all">All Status</option>
          <option value="pending">Pending</option>
          <option value="approved">Approved</option>
          <option value="rejected">Rejected</option>
        </select>
        <input type="month" value={filterMonth} onChange={e=>setFilterMonth(e.target.value)}
          style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"9px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, outline:"none" }}/>
        {filterMonth && <button className="ghost-btn" onClick={()=>setFilterMonth("")} style={{padding:"8px 12px",fontSize:12}}>✕ Clear</button>}
      </div>

      {/* Table */}
      <div className="card" style={{ padding:0, overflow:"hidden" }}>
        {loading ? (
          <div style={{ display:"flex", justifyContent:"center", padding:40 }}><Spinner size={28}/></div>
        ) : (
          <table>
            <thead>
              <tr>
                {isAdmin && <th>Employee</th>}
                <th>Leave Type</th>
                <th>From</th>
                <th>To</th>
                <th>Days</th>
                <th>Reason</th>
                <th>Status</th>
                <th>Actions</th>
              </tr>
            </thead>
            <tbody>
              {displayed.map(l=>(
                <tr key={l.id}>
                  {isAdmin && (
                    <td>
                      <div style={{ display:"flex", alignItems:"center", gap:8 }}>
                        <Avatar name={l.employeeName} size={28}/>
                        <div>
                          <div style={{ fontSize:13, fontWeight:600 }}>{l.employeeName}</div>
                          <div style={{ fontSize:11, color:GRAY }}>{l.department}</div>
                        </div>
                      </div>
                    </td>
                  )}
                  <td><LeaveTypeBadge t={l.leaveType}/></td>
                  <td style={{ color:GOLD, fontSize:13 }}>{l.fromDate}</td>
                  <td style={{ color:GOLD, fontSize:13 }}>{l.toDate}</td>
                  <td style={{ fontWeight:700, color:WHITE }}>{l.days} day{l.days>1?"s":""}</td>
                  <td style={{ color:GRAY, fontSize:13, maxWidth:200 }}>
                    <div style={{ whiteSpace:"nowrap", overflow:"hidden", textOverflow:"ellipsis", maxWidth:180 }}>{l.reason}</div>
                  </td>
                  <td><StatusBadge s={l.status}/></td>
                  <td>
                    <div style={{ display:"flex", gap:6 }}>
                      {isAdmin && l.status==="pending" && <>
                        <button className="ghost-btn" onClick={()=>updateStatus(l,"approved")}
                          style={{ padding:"4px 10px", fontSize:12, borderColor:"#4CC98A", color:"#4CC98A" }}>✅</button>
                        <button className="ghost-btn" onClick={()=>updateStatus(l,"rejected")}
                          style={{ padding:"4px 10px", fontSize:12, borderColor:"#C94C4C", color:"#C94C4C" }}>❌</button>
                      </>}
                      {(!isAdmin && l.status==="pending") && (
                        <button className="danger-btn" onClick={()=>del(l)} style={{ padding:"4px 10px", fontSize:12 }}>Cancel</button>
                      )}
                      {isAdmin && <button className="danger-btn" onClick={()=>del(l)} style={{ padding:"4px 8px" }}>✕</button>}
                    </div>
                  </td>
                </tr>
              ))}
              {displayed.length===0&&!loading&&(
                <tr><td colSpan={isAdmin?8:7} style={{ textAlign:"center", color:GRAY, padding:36 }}>
                  No leave requests found.
                </td></tr>
              )}
            </tbody>
          </table>
        )}
      </div>

      {/* Apply Leave Modal */}
      {modal && (
        <div className="overlay" onClick={()=>setModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{ width:480 }}>
            <div style={{ fontFamily:"'Cinzel',serif", fontSize:18, color:GOLD, marginBottom:6 }}>Apply for Leave</div>
            <div style={{ fontSize:13, color:GRAY, marginBottom:20 }}>Fill in the details below</div>
            <div style={{ display:"flex", flexDirection:"column", gap:14 }}>
              <div>
                <label className="lbl">Leave Type</label>
                <select value={form.leaveType} onChange={e=>setForm({...form,leaveType:e.target.value})}
                  style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}>
                  {LEAVE_TYPES.map(lt=>{
                    const bal = lt.days===999 ? null : getBalance(lt.value, user.joinDate);
                    const isDisabled = bal && (!bal.available || bal.probation);
                    return <option key={lt.value} value={lt.value} disabled={isDisabled}>
                      {lt.label} {bal ? `— ${bal.label}` : "(Unpaid)"}
                      {bal?.probation ? " 🔒" : ""}
                    </option>;
                  })}
                </select>
              </div>
              <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:12 }}>
                <div>
                  <label className="lbl">From Date</label>
                  <input type="date" value={form.fromDate} onChange={e=>setForm({...form,fromDate:e.target.value})}
                    style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                </div>
                <div>
                  <label className="lbl">To Date</label>
                  <input type="date" value={form.toDate} onChange={e=>setForm({...form,toDate:e.target.value})}
                    style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none" }}/>
                </div>
              </div>
              {form.fromDate && form.toDate && (
                <div style={{ background:"rgba(201,168,76,.08)", border:"1px solid rgba(201,168,76,.2)", borderRadius:4, padding:"10px 14px", fontSize:13, color:GOLD }}>
                  📅 Total: <strong>{calcDays(form.fromDate,form.toDate)} day(s)</strong>
                </div>
              )}
              <div>
                <label className="lbl">Reason</label>
                <textarea value={form.reason} onChange={e=>setForm({...form,reason:e.target.value})}
                  placeholder="Explain your reason for leave..."
                  style={{ background:BLACK3, border:"1px solid #444", color:WHITE, padding:"10px 14px", borderRadius:4, fontFamily:"'Rajdhani',sans-serif", fontSize:14, width:"100%", outline:"none", minHeight:80, resize:"vertical" }}/>
              </div>
              {err && <div style={{ background:"rgba(201,76,76,.1)", border:"1px solid rgba(201,76,76,.3)", borderRadius:4, padding:"10px 14px", fontSize:13, color:"#C94C4C" }}>⚠ {err}</div>}
              <div style={{ display:"flex", gap:12, marginTop:6 }}>
                <button className="gold-btn" onClick={applyLeave} disabled={saving} style={{ flex:1 }}>{saving?<Spinner size={14}/>:"Submit Request"}</button>
                <button className="ghost-btn" onClick={()=>setModal(false)} style={{ flex:1 }}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}


// ─── CRM MODULE ───────────────────────────────────────────────────────────────
function CRMPage({ user }) {
  const [leads,      setLeads]      = useState([]);
  const [deals,      setDeals]      = useState([]);
  const [activities, setActivities] = useState([]);
  const [targets,    setTargets]    = useState([]);
  const [loading,    setLoading]    = useState(true);
  const [activeTab,  setActiveTab]  = useState("pipeline");
  const [modal,      setModal]      = useState(false);
  const [actModal,   setActModal]   = useState(false);
  const [targetModal,setTargetModal]= useState(false);
  const [viewing,    setViewing]    = useState(null);
  const [editing,    setEditing]    = useState(null);
  const [saving,     setSaving]     = useState(false);
  const [search,     setSearch]     = useState("");
  const [filterStage,setFilterStage]= useState("all");
  const [filterAssign,setFilterAssign]=useState("all");
  const [dragOver,   setDragOver]   = useState(null);
  const [dragging,   setDragging]   = useState(null);
  const [err,        setErr]        = useState("");

  const LEAD_EMPTY = { name:"", company:"", email:"", phone:"", source:"", stage:"new", value:"", assignedTo:user.name, notes:"", followUpDate:"", priority:"medium", address:"", product:"" };
  const [leadForm, setLeadForm] = useState(LEAD_EMPTY);
  const ACT_EMPTY = { type:"call", notes:"", date: new Date().toISOString().slice(0,10), duration:"", outcome:"", leadId:"" };
  const [actForm, setActForm] = useState(ACT_EMPTY);
  const [targetForm, setTargetForm] = useState({ month: new Date().toISOString().slice(0,7), assignedTo:"", targetAmount:"", targetLeads:"" });

  const LEAD_STAGES = [
    { id:"new",         label:"New Leads",    color:"#4C9AC9", icon:"🆕" },
    { id:"contacted",   label:"Contacted",    color:"#9AC94C", icon:"📞" },
    { id:"qualified",   label:"Qualified",    color:"#C9A84C", icon:"⭐" },
    { id:"proposal",    label:"Proposal",     color:"#C94C9A", icon:"📋" },
    { id:"negotiation", label:"Negotiation",  color:"#C97A4C", icon:"🤝" },
    { id:"won",         label:"Won",          color:"#4CC98A", icon:"✅" },
    { id:"lost",        label:"Lost",         color:"#C94C4C", icon:"❌" },
  ];
  const SOURCES  = ["Walk-in","Referral","Website","Social Media","Cold Call","Email","Job Fair","Advertisement","Other"];
  const ACT_TYPES = ["call","meeting","email","whatsapp","visit","follow-up"];

  useEffect(() => {
    const u1 = onSnapshot(collection(crmDb,"leads"),     snap=>{ setLeads(snap.docs.map(d=>({...d.data(),id:d.id}))); setLoading(false); });
    const u2 = onSnapshot(collection(crmDb,"deals"),     snap=>{ setDeals(snap.docs.map(d=>({...d.data(),id:d.id}))); });
    const u3 = onSnapshot(collection(crmDb,"activities"),snap=>{ setActivities(snap.docs.map(d=>({...d.data(),id:d.id}))); });
    const u4 = onSnapshot(collection(crmDb,"targets"),   snap=>{ setTargets(snap.docs.map(d=>({...d.data(),id:d.id}))); });
    return () => { u1(); u2(); u3(); u4(); };
  }, []);

  // ── Stats ──────────────────────────────────────────────────────────────────
  const totalLeads  = leads.length;
  const wonLeads    = leads.filter(l=>l.stage==="won").length;
  const totalWonVal = leads.filter(l=>l.stage==="won").reduce((a,l)=>a+Number(l.value||0),0);
  const convRate    = totalLeads ? Math.round((wonLeads/totalLeads)*100) : 0;
  const thisMonth   = new Date().toISOString().slice(0,7);
  const monthLeads  = leads.filter(l=>l.createdAt?.seconds && new Date(l.createdAt.seconds*1000).toISOString().slice(0,7)===thisMonth).length;
  const overdue     = leads.filter(l=>l.followUpDate && new Date(l.followUpDate)<new Date() && !["won","lost"].includes(l.stage)).length;

  // ── Helpers ────────────────────────────────────────────────────────────────
  const PKR       = n => `PKR ${Number(n||0).toLocaleString()}`;
  const getLeadActs = id => activities.filter(a=>a.leadId===id).sort((a,b)=>b.date?.localeCompare(a.date));
  const assignees = [...new Set(leads.map(l=>l.assignedTo).filter(Boolean))];

  const PriBadge = ({p}) => {
    const m = {low:["#4C9AC9","Low"],medium:["#C9A84C","Med"],high:["#C94C4C","High"]};
    const [c,l] = m[p]||m.medium;
    return <span style={{fontSize:10,color:c,background:`${c}18`,border:`1px solid ${c}44`,padding:"1px 7px",borderRadius:20,fontWeight:700}}>{l}</span>;
  };

  const StagePill = ({stage}) => {
    const s = LEAD_STAGES.find(x=>x.id===stage)||LEAD_STAGES[0];
    return <span style={{fontSize:11,color:s.color,background:`${s.color}18`,border:`1px solid ${s.color}44`,padding:"2px 10px",borderRadius:20,fontWeight:600}}>{s.icon} {s.label}</span>;
  };

  // ── Save Lead ──────────────────────────────────────────────────────────────
  const saveLead = async () => {
    if (!leadForm.name||!leadForm.phone) { setErr("Name and Phone required."); return; }
    setSaving(true); setErr("");
    const id = editing?.id || Date.now().toString();
    await setDoc(doc(crmDb,"leads",id), { ...leadForm, id, createdBy:user.email, createdAt: editing? leadForm.createdAt : serverTimestamp(), updatedAt:serverTimestamp() });
    setSaving(false); setModal(false); setEditing(null); setLeadForm(LEAD_EMPTY);
  };

  // ── Save Activity ──────────────────────────────────────────────────────────
  const saveActivity = async () => {
    if (!actForm.notes) { setErr("Notes required."); return; }
    setSaving(true);
    const id = Date.now().toString();
    await setDoc(doc(crmDb,"activities",id), { ...actForm, id, addedBy:user.name, createdAt:serverTimestamp() });
    setSaving(false); setActModal(false); setActForm(ACT_EMPTY);
  };

  // ── Save Target ────────────────────────────────────────────────────────────
  const saveTarget = async () => {
    const id = `${targetForm.assignedTo}_${targetForm.month}`;
    await setDoc(doc(crmDb,"targets",id), { ...targetForm, id, setBy:user.email });
    setTargetModal(false);
  };

  // ── Drag & Drop ────────────────────────────────────────────────────────────
  const onDragStart = (e, lead) => { setDragging(lead); e.dataTransfer.effectAllowed = "move"; };
  const onDragOver  = (e, stageId) => { e.preventDefault(); setDragOver(stageId); };
  const onDrop      = async (e, stageId) => {
    e.preventDefault();
    if (dragging && dragging.stage !== stageId) {
      await updateDoc(doc(crmDb,"leads",dragging.id), { stage:stageId, updatedAt:serverTimestamp() });
    }
    setDragging(null); setDragOver(null);
  };

  // ── Filter leads ───────────────────────────────────────────────────────────
  const filtered = leads.filter(l => {
    const s = (l.name||"").toLowerCase().includes(search.toLowerCase()) ||
              (l.company||"").toLowerCase().includes(search.toLowerCase()) ||
              (l.phone||"").toLowerCase().includes(search.toLowerCase());
    const st = filterStage==="all" || l.stage===filterStage;
    const as = filterAssign==="all" || l.assignedTo===filterAssign;
    return s && st && as;
  });

  return (
    <div className="fade">
      {/* ── Header ── */}
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:16}}>
        <div>
          <div style={{fontFamily:"'Cinzel',serif",fontSize:22,color:WHITE}}>Sales CRM</div>
          <div style={{color:GRAY,fontSize:13,marginTop:3}}>
            {overdue>0 && <span style={{color:"#C94C4C",fontWeight:600}}>⚠ {overdue} overdue follow-ups &nbsp;|&nbsp;</span>}
            {monthLeads} new leads this month
          </div>
        </div>
        <div style={{display:"flex",gap:8}}>
          <button className="ghost-btn" onClick={()=>setTargetModal(true)} style={{fontSize:12}}>🎯 Set Target</button>
          <button className="ghost-btn" onClick={()=>{setActForm(ACT_EMPTY);setErr("");setActModal(true);}} style={{fontSize:12}}>📞 Log Activity</button>
          <button className="gold-btn" onClick={()=>{setLeadForm(LEAD_EMPTY);setEditing(null);setErr("");setModal(true);}}>+ New Lead</button>
        </div>
      </div>

      {/* ── Stats Row ── */}
      <div style={{display:"flex",gap:12,flexWrap:"wrap",marginBottom:20}}>
        {[
          {label:"Total Leads",  val:totalLeads,       c:"#4C9AC9", icon:"👥"},
          {label:"Won",          val:wonLeads,          c:"#4CC98A", icon:"✅"},
          {label:"Revenue Won",  val:PKR(totalWonVal),  c:GOLD,      icon:"💰", sm:true},
          {label:"Conv. Rate",   val:`${convRate}%`,    c:"#C94C9A", icon:"📈"},
          {label:"Overdue",      val:overdue,           c:"#C94C4C", icon:"⚠"},
          {label:"This Month",   val:monthLeads,        c:"#9AC94C", icon:"📅"},
        ].map(s=>(
          <div key={s.label} className="card hov" style={{flex:1,minWidth:110}}>
            <div style={{fontSize:11,color:GRAY,letterSpacing:1,textTransform:"uppercase",marginBottom:6}}>{s.icon} {s.label}</div>
            <div style={{fontSize:s.sm?15:24,fontFamily:"'Cinzel',serif",color:s.c,fontWeight:700}}>{s.val}</div>
          </div>
        ))}
      </div>

      {/* ── Tabs ── */}
      <div style={{display:"flex",gap:4,marginBottom:16,borderBottom:`1px solid #222`}}>
        {[["pipeline","🗂 Pipeline"],["leads","📋 All Leads"],["reports","📊 Reports"],["targets","🎯 Targets"]].map(([tab,lbl])=>(
          <div key={tab} onClick={()=>setActiveTab(tab)} style={{
            padding:"9px 18px",cursor:"pointer",fontSize:13,fontWeight:600,
            color:activeTab===tab?GOLD:GRAY,
            borderBottom:activeTab===tab?`2px solid ${GOLD}`:"2px solid transparent",
            transition:"all .2s",
          }}>{lbl}</div>
        ))}
      </div>

      {/* ══════════════════ PIPELINE (Kanban) ══════════════════ */}
      {activeTab==="pipeline" && (
        <div>
          {/* Filters */}
          <div style={{display:"flex",gap:10,marginBottom:14,flexWrap:"wrap"}}>
            <div style={{position:"relative",flex:1,maxWidth:280}}>
              <span style={{position:"absolute",left:10,top:"50%",transform:"translateY(-50%)",color:GRAY,pointerEvents:"none"}}>⌕</span>
              <input style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"8px 10px 8px 30px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}
                placeholder="Search leads..." value={search} onChange={e=>setSearch(e.target.value)}/>
            </div>
            <select value={filterAssign} onChange={e=>setFilterAssign(e.target.value)}
              style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"8px 12px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,outline:"none",cursor:"pointer"}}>
              <option value="all">All Assignees</option>
              {assignees.map(a=><option key={a} value={a}>{a}</option>)}
            </select>
          </div>
          {/* Kanban Board */}
          <div style={{overflowX:"auto",paddingBottom:16}}>
            <div style={{display:"flex",gap:12,minWidth:LEAD_STAGES.length*210}}>
              {LEAD_STAGES.map(stage=>{
                const sLeads = filtered.filter(l=>l.stage===stage.id);
                const sVal   = sLeads.reduce((a,l)=>a+Number(l.value||0),0);
                const isDragTarget = dragOver===stage.id;
                return (
                  <div key={stage.id} style={{flex:1,minWidth:195}}
                    onDragOver={e=>onDragOver(e,stage.id)}
                    onDrop={e=>onDrop(e,stage.id)}
                    onDragLeave={()=>setDragOver(null)}>
                    {/* Column Header */}
                    <div style={{
                      borderTop:`3px solid ${stage.color}`,
                      background:isDragTarget?`${stage.color}15`:BLACK2,
                      borderRadius:"0 0 6px 6px",padding:"10px 12px",marginBottom:8,
                      transition:"background .2s",
                    }}>
                      <div style={{fontSize:13,fontWeight:700,color:stage.color}}>{stage.icon} {stage.label}</div>
                      <div style={{fontSize:11,color:GRAY,marginTop:2}}>{sLeads.length} · {PKR(sVal)}</div>
                    </div>
                    {/* Lead Cards */}
                    <div style={{display:"flex",flexDirection:"column",gap:8,minHeight:100}}>
                      {sLeads.map(lead=>{
                        const leadActs = getLeadActs(lead.id);
                        const isOverdue = lead.followUpDate && new Date(lead.followUpDate)<new Date();
                        return (
                          <div key={lead.id}
                            draggable
                            onDragStart={e=>onDragStart(e,lead)}
                            onClick={()=>setViewing(lead)}
                            style={{
                              background:BLACK2,border:`1px solid ${isOverdue?"#C94C4C":"#222"}`,
                              borderLeft:`4px solid ${stage.color}`,borderRadius:6,padding:10,
                              cursor:"grab",transition:"all .2s",
                            }}>
                            <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start",marginBottom:5}}>
                              <div style={{fontSize:13,fontWeight:700,color:WHITE,flex:1,marginRight:4}}>{lead.name}</div>
                              <PriBadge p={lead.priority}/>
                            </div>
                            {lead.company && <div style={{fontSize:11,color:GRAY,marginBottom:4}}>🏢 {lead.company}</div>}
                            <div style={{fontSize:11,color:GRAY,marginBottom:4}}>📞 {lead.phone}</div>
                            {lead.value && <div style={{fontSize:12,color:GOLD,fontWeight:700,marginBottom:4}}>{PKR(lead.value)}</div>}
                            {lead.assignedTo && <div style={{fontSize:11,color:"#4C9AC9"}}>👤 {lead.assignedTo}</div>}
                            {lead.followUpDate && (
                              <div style={{fontSize:11,color:isOverdue?"#C94C4C":"#4CC98A",marginTop:4,fontWeight:isOverdue?700:400}}>
                                {isOverdue?"⚠ OVERDUE:":"📅"} {lead.followUpDate}
                              </div>
                            )}
                            {leadActs.length>0 && (
                              <div style={{fontSize:10,color:GRAY,marginTop:4,borderTop:"1px solid #1a1a1a",paddingTop:4}}>
                                Last: {leadActs[0].type} — {leadActs[0].date}
                              </div>
                            )}
                          </div>
                        );
                      })}
                      {isDragTarget && (
                        <div style={{border:`2px dashed ${stage.color}`,borderRadius:6,padding:20,textAlign:"center",color:stage.color,fontSize:12,opacity:.7}}>
                          Drop here
                        </div>
                      )}
                      {sLeads.length===0 && !isDragTarget && (
                        <div style={{border:`1px dashed #222`,borderRadius:6,padding:20,textAlign:"center",color:"#333",fontSize:12}}>
                          No leads
                        </div>
                      )}
                    </div>
                  </div>
                );
              })}
            </div>
          </div>
        </div>
      )}

      {/* ══════════════════ ALL LEADS (Table) ══════════════════ */}
      {activeTab==="leads" && (
        <div>
          <div style={{display:"flex",gap:10,marginBottom:14,flexWrap:"wrap"}}>
            <div style={{position:"relative",flex:1,maxWidth:300}}>
              <span style={{position:"absolute",left:10,top:"50%",transform:"translateY(-50%)",color:GRAY,pointerEvents:"none"}}>⌕</span>
              <input style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"8px 10px 8px 30px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}
                placeholder="Search..." value={search} onChange={e=>setSearch(e.target.value)}/>
            </div>
            <select value={filterStage} onChange={e=>setFilterStage(e.target.value)}
              style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"8px 12px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,outline:"none"}}>
              <option value="all">All Stages</option>
              {LEAD_STAGES.map(s=><option key={s.id} value={s.id}>{s.label}</option>)}
            </select>
            <select value={filterAssign} onChange={e=>setFilterAssign(e.target.value)}
              style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"8px 12px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,outline:"none"}}>
              <option value="all">All Assignees</option>
              {assignees.map(a=><option key={a} value={a}>{a}</option>)}
            </select>
          </div>
          <div className="card" style={{padding:0,overflow:"hidden"}}>
            <table>
              <thead><tr><th>Name</th><th>Company</th><th>Phone</th><th>Source</th><th>Value</th><th>Assigned</th><th>Stage</th><th>Follow Up</th><th>Actions</th></tr></thead>
              <tbody>
                {filtered.map(l=>(
                  <tr key={l.id} style={{cursor:"pointer"}} onClick={()=>setViewing(l)}>
                    <td style={{fontWeight:600}}>{l.name}</td>
                    <td style={{color:GRAY,fontSize:13}}>{l.company||"—"}</td>
                    <td style={{fontSize:13}}>{l.phone}</td>
                    <td style={{color:GRAY,fontSize:12}}>{l.source||"—"}</td>
                    <td style={{color:GOLD,fontWeight:600,fontSize:13}}>{l.value?PKR(l.value):"—"}</td>
                    <td style={{color:"#4C9AC9",fontSize:13}}>{l.assignedTo||"—"}</td>
                    <td onClick={e=>e.stopPropagation()}><StagePill stage={l.stage}/></td>
                    <td style={{fontSize:12,color:l.followUpDate&&new Date(l.followUpDate)<new Date()?"#C94C4C":"#4CC98A"}}>{l.followUpDate||"—"}</td>
                    <td onClick={e=>e.stopPropagation()}>
                      <div style={{display:"flex",gap:5}}>
                        <button className="ghost-btn" onClick={()=>{setEditing(l);setLeadForm({...l});setErr("");setModal(true);}} style={{padding:"3px 8px",fontSize:11}}>✏️</button>
                        <button className="ghost-btn" onClick={()=>{setActForm({...ACT_EMPTY,leadId:l.id});setActModal(true);}} style={{padding:"3px 8px",fontSize:11,borderColor:"#4CC98A",color:"#4CC98A"}}>📞</button>
                        <button className="danger-btn" onClick={()=>{if(window.confirm("Delete?"))deleteDoc(doc(crmDb,"leads",l.id));}} style={{padding:"3px 7px",fontSize:11}}>✕</button>
                      </div>
                    </td>
                  </tr>
                ))}
                {filtered.length===0&&<tr><td colSpan={9} style={{textAlign:"center",color:GRAY,padding:32}}>No leads found.</td></tr>}
              </tbody>
            </table>
          </div>
        </div>
      )}

      {/* ══════════════════ REPORTS ══════════════════ */}
      {activeTab==="reports" && (
        <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:20}}>
          {/* Stage breakdown */}
          <div className="card">
            <div style={{fontFamily:"'Cinzel',serif",fontSize:13,color:GOLD,letterSpacing:1,marginBottom:16}}>LEADS BY STAGE</div>
            {LEAD_STAGES.map(s=>{
              const count = leads.filter(l=>l.stage===s.id).length;
              const pct   = totalLeads ? Math.round((count/totalLeads)*100) : 0;
              return (
                <div key={s.id} style={{marginBottom:12}}>
                  <div style={{display:"flex",justifyContent:"space-between",marginBottom:4}}>
                    <span style={{fontSize:13,color:WHITE}}>{s.icon} {s.label}</span>
                    <span style={{fontSize:13,color:s.color,fontWeight:600}}>{count} ({pct}%)</span>
                  </div>
                  <div style={{background:"#1a1a1a",borderRadius:4,height:6}}>
                    <div style={{background:s.color,borderRadius:4,height:6,width:`${pct}%`,transition:"width .3s"}}/>
                  </div>
                </div>
              );
            })}
          </div>
          {/* Source breakdown */}
          <div className="card">
            <div style={{fontFamily:"'Cinzel',serif",fontSize:13,color:GOLD,letterSpacing:1,marginBottom:16}}>LEADS BY SOURCE</div>
            {SOURCES.map(src=>{
              const count = leads.filter(l=>l.source===src).length;
              if(!count) return null;
              const pct = totalLeads ? Math.round((count/totalLeads)*100) : 0;
              return (
                <div key={src} style={{marginBottom:10}}>
                  <div style={{display:"flex",justifyContent:"space-between",marginBottom:3}}>
                    <span style={{fontSize:13,color:WHITE}}>{src}</span>
                    <span style={{fontSize:13,color:GOLD,fontWeight:600}}>{count} ({pct}%)</span>
                  </div>
                  <div style={{background:"#1a1a1a",borderRadius:4,height:5}}>
                    <div style={{background:GOLD,borderRadius:4,height:5,width:`${pct}%`}}/>
                  </div>
                </div>
              );
            })}
          </div>
          {/* Top performers */}
          <div className="card">
            <div style={{fontFamily:"'Cinzel',serif",fontSize:13,color:GOLD,letterSpacing:1,marginBottom:16}}>TOP PERFORMERS</div>
            {assignees.map(a=>{
              const aLeads = leads.filter(l=>l.assignedTo===a);
              const aWon   = aLeads.filter(l=>l.stage==="won").length;
              const aVal   = aLeads.filter(l=>l.stage==="won").reduce((x,l)=>x+Number(l.value||0),0);
              return (
                <div key={a} style={{display:"flex",alignItems:"center",gap:10,marginBottom:14}}>
                  <Avatar name={a} size={32}/>
                  <div style={{flex:1}}>
                    <div style={{fontSize:13,color:WHITE,fontWeight:600}}>{a}</div>
                    <div style={{fontSize:11,color:GRAY}}>{aLeads.length} leads · {aWon} won</div>
                  </div>
                  <div style={{fontSize:13,color:GOLD,fontWeight:700}}>{PKR(aVal)}</div>
                </div>
              );
            })}
            {assignees.length===0&&<div style={{color:GRAY,fontSize:13}}>No data yet.</div>}
          </div>
          {/* Recent activities */}
          <div className="card">
            <div style={{fontFamily:"'Cinzel',serif",fontSize:13,color:GOLD,letterSpacing:1,marginBottom:16}}>RECENT ACTIVITIES</div>
            {activities.slice(0,8).map(a=>{
              const lead = leads.find(l=>l.id===a.leadId);
              return (
                <div key={a.id} style={{display:"flex",gap:10,marginBottom:12,alignItems:"flex-start"}}>
                  <div style={{width:32,height:32,borderRadius:"50%",background:"rgba(201,76,76,.15)",border:"1px solid rgba(201,76,76,.3)",display:"flex",alignItems:"center",justifyContent:"center",fontSize:14,flexShrink:0}}>
                    {a.type==="call"?"📞":a.type==="meeting"?"🤝":a.type==="email"?"📧":a.type==="whatsapp"?"💬":"📋"}
                  </div>
                  <div style={{flex:1}}>
                    <div style={{fontSize:13,color:WHITE,fontWeight:500}}>{a.type.charAt(0).toUpperCase()+a.type.slice(1)} — {lead?.name||"Unknown"}</div>
                    <div style={{fontSize:12,color:GRAY}}>{a.notes}</div>
                    <div style={{fontSize:11,color:GRAY,marginTop:2}}>{a.date} · by {a.addedBy}</div>
                  </div>
                </div>
              );
            })}
            {activities.length===0&&<div style={{color:GRAY,fontSize:13}}>No activities yet.</div>}
          </div>
        </div>
      )}

      {/* ══════════════════ TARGETS ══════════════════ */}
      {activeTab==="targets" && (
        <div>
          <div style={{display:"grid",gridTemplateColumns:"repeat(auto-fill,minmax(260px,1fr))",gap:16}}>
            {targets.filter(t=>t.month===thisMonth).map(t=>{
              const tLeads  = leads.filter(l=>l.assignedTo===t.assignedTo && l.createdAt?.seconds && new Date(l.createdAt.seconds*1000).toISOString().slice(0,7)===t.month);
              const tWonVal = tLeads.filter(l=>l.stage==="won").reduce((a,l)=>a+Number(l.value||0),0);
              const tWonCnt = tLeads.filter(l=>l.stage==="won").length;
              const valPct  = t.targetAmount ? Math.min(100,Math.round((tWonVal/Number(t.targetAmount))*100)) : 0;
              const cntPct  = t.targetLeads  ? Math.min(100,Math.round((tWonCnt/Number(t.targetLeads))*100))  : 0;
              return (
                <div key={t.id} className="card" style={{borderTop:`3px solid ${valPct>=100?"#4CC98A":GOLD}`}}>
                  <div style={{display:"flex",alignItems:"center",gap:10,marginBottom:14}}>
                    <Avatar name={t.assignedTo} size={36}/>
                    <div>
                      <div style={{fontSize:14,fontWeight:700,color:WHITE}}>{t.assignedTo}</div>
                      <div style={{fontSize:12,color:GRAY}}>{t.month}</div>
                    </div>
                  </div>
                  <div style={{marginBottom:12}}>
                    <div style={{display:"flex",justifyContent:"space-between",marginBottom:4}}>
                      <span style={{fontSize:12,color:GRAY}}>Revenue</span>
                      <span style={{fontSize:12,color:valPct>=100?"#4CC98A":GOLD,fontWeight:600}}>{PKR(tWonVal)} / {PKR(t.targetAmount)}</span>
                    </div>
                    <div style={{background:"#1a1a1a",borderRadius:4,height:8}}>
                      <div style={{background:valPct>=100?"#4CC98A":GOLD,borderRadius:4,height:8,width:`${valPct}%`,transition:"width .3s"}}/>
                    </div>
                    <div style={{fontSize:11,color:GRAY,marginTop:3}}>{valPct}% achieved</div>
                  </div>
                  <div>
                    <div style={{display:"flex",justifyContent:"space-between",marginBottom:4}}>
                      <span style={{fontSize:12,color:GRAY}}>Leads Won</span>
                      <span style={{fontSize:12,color:cntPct>=100?"#4CC98A":"#4C9AC9",fontWeight:600}}>{tWonCnt} / {t.targetLeads}</span>
                    </div>
                    <div style={{background:"#1a1a1a",borderRadius:4,height:8}}>
                      <div style={{background:cntPct>=100?"#4CC98A":"#4C9AC9",borderRadius:4,height:8,width:`${cntPct}%`}}/>
                    </div>
                  </div>
                </div>
              );
            })}
            {targets.filter(t=>t.month===thisMonth).length===0 && (
              <div style={{color:GRAY,fontSize:14,padding:20}}>No targets set for this month. Click "Set Target" to add.</div>
            )}
          </div>
        </div>
      )}

      {/* ══════════════════ LEAD DETAIL MODAL ══════════════════ */}
      {viewing && (
        <div className="overlay" onClick={()=>setViewing(null)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{width:560,maxHeight:"90vh",overflowY:"auto"}}>
            <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:16}}>
              <div style={{fontFamily:"'Cinzel',serif",fontSize:18,color:GOLD}}>Lead Details</div>
              <button onClick={()=>setViewing(null)} style={{background:"none",border:"none",color:GRAY,cursor:"pointer",fontSize:20}}>✕</button>
            </div>
            <div style={{display:"flex",alignItems:"center",gap:12,marginBottom:16,padding:14,background:BLACK3,borderRadius:6}}>
              <Avatar name={viewing.name} size={46}/>
              <div style={{flex:1}}>
                <div style={{fontSize:16,fontWeight:700,color:WHITE}}>{viewing.name}</div>
                <div style={{fontSize:13,color:GRAY}}>{viewing.company||"—"}</div>
                <div style={{display:"flex",gap:8,marginTop:6,flexWrap:"wrap"}}>
                  <StagePill stage={viewing.stage}/>
                  <PriBadge p={viewing.priority}/>
                </div>
              </div>
              <div style={{textAlign:"right"}}>
                <div style={{fontSize:18,color:GOLD,fontFamily:"'Cinzel',serif",fontWeight:700}}>{viewing.value?PKR(viewing.value):"—"}</div>
                <div style={{fontSize:11,color:GRAY}}>Deal Value</div>
              </div>
            </div>
            {/* Quick stage change */}
            <div style={{marginBottom:16}}>
              <label className="lbl">Move to Stage</label>
              <div style={{display:"flex",gap:6,flexWrap:"wrap"}}>
                {LEAD_STAGES.map(s=>(
                  <button key={s.id} onClick={()=>updateDoc(doc(crmDb,"leads",viewing.id),{stage:s.id,updatedAt:serverTimestamp()})}
                    style={{background:viewing.stage===s.id?s.color:"transparent",border:`1px solid ${s.color}`,color:viewing.stage===s.id?BLACK:s.color,padding:"5px 12px",borderRadius:4,cursor:"pointer",fontFamily:"'Rajdhani',sans-serif",fontSize:12,fontWeight:600,transition:"all .2s"}}>
                    {s.icon} {s.label}
                  </button>
                ))}
              </div>
            </div>
            {/* Details */}
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:4,marginBottom:16}}>
              {[["Phone",viewing.phone],["Email",viewing.email||"—"],["Source",viewing.source||"—"],["Assigned To",viewing.assignedTo||"—"],["Follow Up",viewing.followUpDate||"—"],["Product",viewing.product||"—"]].map(([k,v])=>(
                <div key={k} style={{padding:"8px 0",borderBottom:"1px solid #1a1a1a"}}>
                  <div style={{fontSize:11,color:GRAY,marginBottom:2}}>{k}</div>
                  <div style={{fontSize:13,color:WHITE}}>{v}</div>
                </div>
              ))}
            </div>
            {viewing.notes && <div style={{background:BLACK3,borderRadius:6,padding:12,marginBottom:16,fontSize:13,color:GRAY}}>📝 {viewing.notes}</div>}
            {/* Activities */}
            <div style={{marginBottom:16}}>
              <div style={{fontFamily:"'Cinzel',serif",fontSize:12,color:GOLD,letterSpacing:1,marginBottom:10}}>ACTIVITY LOG</div>
              {getLeadActs(viewing.id).map(a=>(
                <div key={a.id} style={{display:"flex",gap:10,marginBottom:10,padding:"8px",background:BLACK3,borderRadius:4}}>
                  <span style={{fontSize:16}}>{a.type==="call"?"📞":a.type==="meeting"?"🤝":a.type==="email"?"📧":a.type==="whatsapp"?"💬":"📋"}</span>
                  <div style={{flex:1}}>
                    <div style={{fontSize:12,color:WHITE,fontWeight:600}}>{a.type.charAt(0).toUpperCase()+a.type.slice(1)} — {a.date}</div>
                    <div style={{fontSize:12,color:GRAY}}>{a.notes}</div>
                    {a.outcome&&<div style={{fontSize:11,color:"#4CC98A",marginTop:2}}>Result: {a.outcome}</div>}
                  </div>
                </div>
              ))}
              {getLeadActs(viewing.id).length===0&&<div style={{color:GRAY,fontSize:12}}>No activities yet.</div>}
            </div>
            <div style={{display:"flex",gap:10}}>
              <button className="gold-btn" onClick={()=>{setEditing(viewing);setLeadForm({...viewing});setViewing(null);setModal(true);}} style={{flex:1}}>✏️ Edit</button>
              <button className="ghost-btn" onClick={()=>{setActForm({...ACT_EMPTY,leadId:viewing.id});setActModal(true);}} style={{flex:1,borderColor:"#4CC98A",color:"#4CC98A"}}>📞 Log Activity</button>
              <button className="ghost-btn" onClick={()=>setViewing(null)} style={{flex:1}}>Close</button>
            </div>
          </div>
        </div>
      )}

      {/* ══════════════════ ADD/EDIT LEAD MODAL ══════════════════ */}
      {modal && (
        <div className="overlay" onClick={()=>setModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{width:560,maxHeight:"90vh",overflowY:"auto"}}>
            <div style={{fontFamily:"'Cinzel',serif",fontSize:18,color:GOLD,marginBottom:20}}>{editing?"Edit Lead":"New Lead"}</div>
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:14}}>
              {[["Client Name","name","text","John Doe"],["Company","company","text","Company Ltd"],["Phone","phone","text","+92 300 0000000"],["Email","email","email","john@co.com"],["Deal Value (PKR)","value","number","50000"],["Product/Service","product","text","e.g. Premium Plan"],["Assigned To","assignedTo","text","Employee name"],["Follow Up Date","followUpDate","date",""]].map(([label,key,type,ph])=>(
                <div key={key}>
                  <label className="lbl">{label}</label>
                  <input type={type} placeholder={ph} value={leadForm[key]||""} onChange={e=>setLeadForm({...leadForm,[key]:e.target.value})}
                    style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
                </div>
              ))}
              <div>
                <label className="lbl">Source</label>
                <select value={leadForm.source} onChange={e=>setLeadForm({...leadForm,source:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}>
                  <option value="">— Select —</option>
                  {SOURCES.map(s=><option key={s} value={s}>{s}</option>)}
                </select>
              </div>
              <div>
                <label className="lbl">Stage</label>
                <select value={leadForm.stage} onChange={e=>setLeadForm({...leadForm,stage:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}>
                  {LEAD_STAGES.map(s=><option key={s.id} value={s.id}>{s.label}</option>)}
                </select>
              </div>
              <div>
                <label className="lbl">Priority</label>
                <select value={leadForm.priority} onChange={e=>setLeadForm({...leadForm,priority:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}>
                  <option value="low">Low</option>
                  <option value="medium">Medium</option>
                  <option value="high">High</option>
                </select>
              </div>
              <div style={{gridColumn:"1/-1"}}>
                <label className="lbl">Notes</label>
                <textarea value={leadForm.notes||""} onChange={e=>setLeadForm({...leadForm,notes:e.target.value})} placeholder="Notes about this lead..."
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none",minHeight:70,resize:"vertical"}}/>
              </div>
            </div>
            {err&&<div style={{background:"rgba(201,76,76,.1)",border:"1px solid rgba(201,76,76,.3)",borderRadius:4,padding:"10px 14px",fontSize:13,color:"#C94C4C",marginTop:12}}>⚠ {err}</div>}
            <div style={{display:"flex",gap:12,marginTop:16}}>
              <button className="gold-btn" onClick={saveLead} disabled={saving} style={{flex:1}}>{saving?<Spinner size={14}/>:(editing?"Save Changes":"Add Lead")}</button>
              <button className="ghost-btn" onClick={()=>setModal(false)} style={{flex:1}}>Cancel</button>
            </div>
          </div>
        </div>
      )}

      {/* ══════════════════ LOG ACTIVITY MODAL ══════════════════ */}
      {actModal && (
        <div className="overlay" onClick={()=>setActModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{width:440}}>
            <div style={{fontFamily:"'Cinzel',serif",fontSize:18,color:GOLD,marginBottom:20}}>Log Activity</div>
            <div style={{display:"flex",flexDirection:"column",gap:14}}>
              <div>
                <label className="lbl">Lead</label>
                <select value={actForm.leadId} onChange={e=>setActForm({...actForm,leadId:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}>
                  <option value="">— Select Lead —</option>
                  {leads.map(l=><option key={l.id} value={l.id}>{l.name} — {l.company||l.phone}</option>)}
                </select>
              </div>
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:12}}>
                <div>
                  <label className="lbl">Activity Type</label>
                  <select value={actForm.type} onChange={e=>setActForm({...actForm,type:e.target.value})}
                    style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}>
                    {ACT_TYPES.map(t=><option key={t} value={t}>{t.charAt(0).toUpperCase()+t.slice(1)}</option>)}
                  </select>
                </div>
                <div>
                  <label className="lbl">Date</label>
                  <input type="date" value={actForm.date} onChange={e=>setActForm({...actForm,date:e.target.value})}
                    style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
                </div>
              </div>
              <div>
                <label className="lbl">Notes / Summary</label>
                <textarea value={actForm.notes} onChange={e=>setActForm({...actForm,notes:e.target.value})} placeholder="What happened in this activity?"
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none",minHeight:80,resize:"vertical"}}/>
              </div>
              <div>
                <label className="lbl">Outcome (optional)</label>
                <input value={actForm.outcome||""} onChange={e=>setActForm({...actForm,outcome:e.target.value})} placeholder="e.g. Interested, Call back tomorrow"
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              {err&&<div style={{background:"rgba(201,76,76,.1)",border:"1px solid rgba(201,76,76,.3)",borderRadius:4,padding:"10px 14px",fontSize:13,color:"#C94C4C"}}>⚠ {err}</div>}
              <div style={{display:"flex",gap:12}}>
                <button className="gold-btn" onClick={saveActivity} disabled={saving} style={{flex:1}}>{saving?<Spinner size={14}/>:"Save Activity"}</button>
                <button className="ghost-btn" onClick={()=>setActModal(false)} style={{flex:1}}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}

      {/* ══════════════════ SET TARGET MODAL ══════════════════ */}
      {targetModal && (
        <div className="overlay" onClick={()=>setTargetModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{width:420}}>
            <div style={{fontFamily:"'Cinzel',serif",fontSize:18,color:GOLD,marginBottom:20}}>Set Sales Target</div>
            <div style={{display:"flex",flexDirection:"column",gap:14}}>
              <div>
                <label className="lbl">Employee</label>
                <input value={targetForm.assignedTo} onChange={e=>setTargetForm({...targetForm,assignedTo:e.target.value})} placeholder="Employee name"
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              <div>
                <label className="lbl">Month</label>
                <input type="month" value={targetForm.month} onChange={e=>setTargetForm({...targetForm,month:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              <div>
                <label className="lbl">Revenue Target (PKR)</label>
                <input type="number" value={targetForm.targetAmount} onChange={e=>setTargetForm({...targetForm,targetAmount:e.target.value})} placeholder="e.g. 500000"
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              <div>
                <label className="lbl">Leads Target (count)</label>
                <input type="number" value={targetForm.targetLeads} onChange={e=>setTargetForm({...targetForm,targetLeads:e.target.value})} placeholder="e.g. 20"
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              <div style={{display:"flex",gap:12}}>
                <button className="gold-btn" onClick={saveTarget} style={{flex:1}}>Save Target</button>
                <button className="ghost-btn" onClick={()=>setTargetModal(false)} style={{flex:1}}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}


// ─── SALES & COMMISSION MODULE (Per-Department Tiers) ─────────────────────────
const DEFAULT_TIERS = [
  { id:"t1", minSales:12, maxSales:14, baseSalary: 45000, punctuality: 5000, label:"12 Sales" },
  { id:"t2", minSales:15, maxSales:17, baseSalary: 50000, punctuality: 5000, label:"15 Sales" },
  { id:"t3", minSales:18, maxSales:20, baseSalary: 67000, punctuality: 5000, label:"18 Sales" },
  { id:"t4", minSales:21, maxSales:24, baseSalary: 75000, punctuality: 5000, label:"21 Sales" },
  { id:"t5", minSales:25, maxSales:27, baseSalary: 90000, punctuality:10000, label:"25 Sales" },
  { id:"t6", minSales:28, maxSales:31, baseSalary:105000, punctuality:10000, label:"28 Sales" },
  { id:"t7", minSales:32, maxSales:39, baseSalary:110000, punctuality:15000, label:"32 Sales" },
  { id:"t8", minSales:40, maxSales:44, baseSalary:125000, punctuality:25000, label:"40 Sales" },
  { id:"t9", minSales:45, maxSales:54, baseSalary:132000, punctuality:25000, label:"45 Sales" },
  { id:"t10",minSales:55, maxSales:999,baseSalary:160000, punctuality:25000, label:"55+ Sales", perExtraSale:2000 },
];

const calcSalaryFromSales = (salesCount, tiers) => {
  if (!tiers || tiers.length===0) return null;
  const sorted = [...tiers].sort((a,b)=>a.minSales-b.minSales);
  // Find matching tier
  let tier = sorted.find(t => salesCount >= t.minSales && salesCount <= t.maxSales);
  if (!tier) {
    // Check if above highest tier with perExtraSale
    const highest = sorted[sorted.length-1];
    if (highest && salesCount > highest.maxSales && highest.perExtraSale) tier = highest;
  }
  if (!tier) return { base:0, punctuality:0, total:0, tier:{label:"Below Minimum"}, belowMin:true };
  let base = tier.baseSalary;
  if (tier.perExtraSale && salesCount > tier.minSales) {
    const extra = Math.max(0, salesCount - (tier.maxSales<999?tier.maxSales:tier.minSales));
    if (tier.maxSales>=999 && salesCount>tier.minSales) base += (salesCount-tier.minSales)*tier.perExtraSale;
  }
  return { base, punctuality: tier.punctuality, total: base+tier.punctuality, tier };
};

function SalesPage({ user }) {
  const [salesRecords, setSalesRecords] = useState([]);
  const [employees,    setEmployees]    = useState([]);
  const [departments,  setDepartments]  = useState([]);
  const [deptTiers,    setDeptTiers]    = useState({}); // { deptName: [tiers] }
  const [loading,      setLoading]      = useState(true);
  const [modal,        setModal]        = useState(false);
  const [tierModal,    setTierModal]    = useState(false);
  const [tierDept,     setTierDept]     = useState(""); // which dept's tiers being edited
  const [saving,       setSaving]       = useState(false);
  const [filterMonth,  setFilterMonth]  = useState(new Date().toISOString().slice(0,7));
  const [viewDept,     setViewDept]     = useState("all");
  const [form,         setForm]         = useState({ employeeEmail:"", salesCount:"", month: new Date().toISOString().slice(0,7), notes:"" });
  const [editingTier,  setEditingTier]  = useState(null);
  const [tierForm,     setTierForm]     = useState({ minSales:"", maxSales:"", baseSalary:"", punctuality:"", label:"", perExtraSale:"" });

  const isAdmin = ["super_admin","ceo","manager","hr_manager","finance_manager"].includes(user.role);

  useEffect(()=>{
    const u1 = onSnapshot(collection(db,"sales_records"), snap=>{
      setSalesRecords(snap.docs.map(d=>({...d.data(),id:d.id})));
      setLoading(false);
    });
    const u2 = onSnapshot(collection(db,"employees"), snap=>setEmployees(snap.docs.map(d=>({...d.data(),id:d.id}))));
    const u3 = onSnapshot(collection(db,"departments"), snap=>setDepartments(snap.docs.map(d=>({...d.data(),id:d.id}))));
    // Load per-department tiers
    const u4 = onSnapshot(collection(db,"dept_salary_tiers"), snap=>{
      const map = {};
      snap.docs.forEach(d=>{ map[d.id] = d.data().tiers || []; });
      setDeptTiers(map);
    });
    return ()=>{ u1(); u2(); u3(); u4(); };
  },[]);

  const PKR = n => `PKR ${Number(n||0).toLocaleString()}`;

  // Departments that HAVE a tier system (commission-based)
  const commissionDepts = departments.filter(d => deptTiers[d.name] && deptTiers[d.name].length>0);
  // Get tiers for a department (or empty)
  const getTiersFor = (deptName) => deptTiers[deptName] || [];
  const deptHasTiers = (deptName) => getTiersFor(deptName).length > 0;

  const monthRecords = salesRecords.filter(r=>r.month===filterMonth && (viewDept==="all"||r.department===viewDept));

  // ── Save sales entry ──────────────────────────────────────────────────────
  const saveSales = async () => {
    if (!form.employeeEmail || form.salesCount==="") return;
    setSaving(true);
    const sales = Number(form.salesCount);
    const emp = employees.find(e=>(e.email||e.id)===form.employeeEmail);
    const deptTierList = getTiersFor(emp?.department);
    const calc = calcSalaryFromSales(sales, deptTierList);
    const id = `${form.employeeEmail}_${form.month}`;

    await setDoc(doc(db,"sales_records",id),{
      employeeEmail: form.employeeEmail,
      employeeName:  emp?.name||"",
      department:    emp?.department||"",
      month:         form.month,
      salesCount:    sales,
      tierLabel:     calc?.tier?.label||"—",
      baseSalary:    calc?.base||0,
      punctualityBonus: calc?.punctuality||0,
      totalSalary:   calc?.total||0,
      notes:         form.notes||"",
      enteredBy:     user.email,
      updatedAt:     serverTimestamp(),
    });

    // Auto-update payroll
    await setDoc(doc(db,"payroll",`${form.employeeEmail}_${form.month}`),{
      employeeEmail: form.employeeEmail,
      employeeName:  emp?.name||"",
      department:    emp?.department||"",
      basicSalary:   calc?.base||0,
      punctualityBonus: calc?.punctuality||0,
      grossSalary:   calc?.total||0,
      allowances:0, deductions:0, fines:0, absentDays:0, absentDeduction:0, tax:0,
      netSalary:     calc?.total||0,
      month:         form.month,
      status:        "pending",
      salesCount:    sales,
      tierLabel:     calc?.tier?.label||"—",
      notes:`Auto-calculated from ${sales} sales — Tier: ${calc?.tier?.label||"N/A"}`,
      generatedAt:   serverTimestamp(),
    },{ merge:true });

    setSaving(false); setModal(false);
    setForm({ employeeEmail:"", salesCount:"", month:filterMonth, notes:"" });
  };

  // ── Tier CRUD ──────────────────────────────────────────────────────────────
  const openTierManager = (deptName) => {
    setTierDept(deptName);
    setTierForm({ minSales:"", maxSales:"", baseSalary:"", punctuality:"", label:"", perExtraSale:"" });
    setEditingTier(null);
    setTierModal(true);
  };

  const initDeptWithDefaults = async (deptName) => {
    await setDoc(doc(db,"dept_salary_tiers",deptName),{ tiers: DEFAULT_TIERS, updatedAt: serverTimestamp() });
  };

  const addOrUpdateTier = async () => {
    if (!tierForm.minSales || !tierForm.baseSalary) return;
    const currentTiers = getTiersFor(tierDept);
    const newTier = {
      id: editingTier?.id || `t_${Date.now()}`,
      minSales: Number(tierForm.minSales),
      maxSales: Number(tierForm.maxSales)||999,
      baseSalary: Number(tierForm.baseSalary),
      punctuality: Number(tierForm.punctuality)||0,
      label: tierForm.label || `${tierForm.minSales} Sales`,
      ...(tierForm.perExtraSale ? { perExtraSale: Number(tierForm.perExtraSale) } : {}),
    };
    let updated;
    if (editingTier) {
      updated = currentTiers.map(t => t.id===editingTier.id ? newTier : t);
    } else {
      updated = [...currentTiers, newTier];
    }
    updated.sort((a,b)=>a.minSales-b.minSales);
    await setDoc(doc(db,"dept_salary_tiers",tierDept),{ tiers: updated, updatedAt: serverTimestamp() });
    setTierForm({ minSales:"", maxSales:"", baseSalary:"", punctuality:"", label:"", perExtraSale:"" });
    setEditingTier(null);
  };

  const editTier = (tier) => {
    setEditingTier(tier);
    setTierForm({
      minSales: tier.minSales, maxSales: tier.maxSales===999?"":tier.maxSales,
      baseSalary: tier.baseSalary, punctuality: tier.punctuality,
      label: tier.label, perExtraSale: tier.perExtraSale||"",
    });
  };

  const deleteTier = async (tierId) => {
    if (!window.confirm("Delete this tier?")) return;
    const updated = getTiersFor(tierDept).filter(t=>t.id!==tierId);
    await setDoc(doc(db,"dept_salary_tiers",tierDept),{ tiers: updated, updatedAt: serverTimestamp() });
  };

  const removeTierSystemFromDept = async () => {
    if (!window.confirm(`Remove entire commission system from ${tierDept}? This department will go back to fixed salary.`)) return;
    await deleteDoc(doc(db,"dept_salary_tiers",tierDept));
    setTierModal(false);
  };

  // Stats
  const totalSales  = monthRecords.reduce((a,r)=>a+Number(r.salesCount||0),0);
  const totalPayout = monthRecords.reduce((a,r)=>a+Number(r.totalSalary||0),0);
  const topPerformer= [...monthRecords].sort((a,b)=>b.salesCount-a.salesCount)[0];
  const avgSales    = monthRecords.length ? Math.round(totalSales/monthRecords.length) : 0;

  return (
    <div className="fade">
      {/* Header */}
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:20}}>
        <div>
          <div style={{fontFamily:"'Cinzel',serif",fontSize:22,color:WHITE}}>Sales & Commission</div>
          <div style={{color:GRAY,fontSize:13,marginTop:3}}>Each department has its own salary tier structure</div>
        </div>
        {isAdmin && <button className="gold-btn" onClick={()=>{setForm({employeeEmail:"",salesCount:"",month:filterMonth,notes:""});setModal(true);}}>+ Enter Sales</button>}
      </div>

      {/* Stats */}
      <div style={{display:"flex",gap:14,flexWrap:"wrap",marginBottom:20}}>
        {[
          {label:"Total Sales",      val:totalSales,       c:"#4C9AC9", icon:"📊"},
          {label:"Avg Per Employee", val:avgSales,         c:GOLD,      icon:"📈"},
          {label:"Total Payout",     val:PKR(totalPayout), c:"#4CC98A", icon:"💰", sm:true},
          {label:"Top Performer",    val:topPerformer?.employeeName||"—", c:"#C94C9A", icon:"🏆", sm:true},
        ].map(s=>(
          <div key={s.label} className="card hov" style={{flex:1,minWidth:130}}>
            <div style={{fontSize:11,color:GRAY,letterSpacing:1,textTransform:"uppercase",marginBottom:8}}>{s.icon} {s.label}</div>
            <div style={{fontSize:s.sm?14:24,fontFamily:"'Cinzel',serif",color:s.c,fontWeight:700}}>{s.val}</div>
          </div>
        ))}
      </div>

      {/* Filters */}
      <div style={{display:"flex",gap:12,marginBottom:16,alignItems:"center",flexWrap:"wrap"}}>
        <input type="month" value={filterMonth} onChange={e=>setFilterMonth(e.target.value)}
          style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"9px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,outline:"none"}}/>
        <select value={viewDept} onChange={e=>setViewDept(e.target.value)}
          style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"9px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,outline:"none",cursor:"pointer"}}>
          <option value="all">All Departments</option>
          {departments.map(d=><option key={d.id} value={d.name}>{d.name}</option>)}
        </select>
      </div>

      {/* Per-Department Tier Cards */}
      <div style={{display:"grid",gridTemplateColumns:"repeat(auto-fill,minmax(280px,1fr))",gap:16,marginBottom:24}}>
        {departments.map(dept=>{
          const hasTiers = deptHasTiers(dept.name);
          const tierList = getTiersFor(dept.name);
          return (
            <div key={dept.id} className="card" style={{borderTop:`3px solid ${hasTiers?dept.color||GOLD:"#444"}`}}>
              <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start",marginBottom:10}}>
                <div>
                  <div style={{fontFamily:"'Cinzel',serif",fontSize:14,color:WHITE,fontWeight:700}}>{dept.name}</div>
                  <div style={{fontSize:11,color:GRAY,marginTop:2}}>{hasTiers?`${tierList.length} tiers configured`:"No commission system"}</div>
                </div>
                {isAdmin && (
                  hasTiers ? (
                    <button className="ghost-btn" onClick={()=>openTierManager(dept.name)} style={{padding:"4px 10px",fontSize:11}}>⚙️ Manage</button>
                  ) : (
                    <button className="ghost-btn" onClick={async ()=>{await initDeptWithDefaults(dept.name); openTierManager(dept.name);}} style={{padding:"4px 10px",fontSize:11,borderColor:"#4CC98A",color:"#4CC98A"}}>+ Setup Tiers</button>
                  )
                )}
              </div>
              {hasTiers ? (
                <div style={{maxHeight:140,overflowY:"auto"}}>
                  {tierList.map(t=>(
                    <div key={t.id} style={{display:"flex",justifyContent:"space-between",padding:"5px 0",borderBottom:"1px solid #1a1a1a",fontSize:12}}>
                      <span style={{color:GOLD}}>{t.label}</span>
                      <span style={{color:WHITE,fontWeight:600}}>{PKR(t.baseSalary+t.punctuality)}{t.perExtraSale?"+":""}</span>
                    </div>
                  ))}
                </div>
              ) : (
                <div style={{fontSize:12,color:GRAY,fontStyle:"italic"}}>Fixed salary department (e.g. Operations) — no sales-based pay</div>
              )}
            </div>
          );
        })}
        {departments.length===0&&<div style={{color:GRAY,fontSize:13}}>No departments found. Add departments first.</div>}
      </div>

      {/* Monthly Sales Records Table */}
      <div className="card" style={{padding:0,overflow:"hidden"}}>
        <div style={{fontFamily:"'Cinzel',serif",fontSize:13,color:GOLD,letterSpacing:1,padding:"14px 16px",borderBottom:"1px solid #222"}}>
          {filterMonth} — SALES RECORDS
        </div>
        {loading?<div style={{display:"flex",justifyContent:"center",padding:40}}><Spinner size={28}/></div>:(
          <table>
            <thead><tr><th>Employee</th><th>Department</th><th>Sales</th><th>Tier</th><th>Base</th><th>Punctuality</th><th>Total Salary</th></tr></thead>
            <tbody>
              {monthRecords.sort((a,b)=>b.salesCount-a.salesCount).map(r=>(
                <tr key={r.id}>
                  <td><div style={{display:"flex",alignItems:"center",gap:8}}><Avatar name={r.employeeName} size={26}/><span style={{fontSize:13,fontWeight:600}}>{r.employeeName}</span></div></td>
                  <td style={{color:GRAY,fontSize:13}}>{r.department}</td>
                  <td><span style={{fontSize:16,fontFamily:"'Cinzel',serif",color:r.salesCount>=55?"#4CC98A":r.salesCount>=25?GOLD:"#4C9AC9",fontWeight:700}}>{r.salesCount}</span></td>
                  <td><span style={{fontSize:11,color:GOLD,background:"rgba(201,168,76,.1)",border:"1px solid rgba(201,168,76,.2)",padding:"2px 8px",borderRadius:20,fontWeight:600}}>{r.tierLabel}</span></td>
                  <td style={{fontSize:13}}>{PKR(r.baseSalary)}</td>
                  <td style={{fontSize:13,color:"#4CC98A"}}>{PKR(r.punctualityBonus)}</td>
                  <td style={{fontWeight:700,color:WHITE}}>{PKR(r.totalSalary)}</td>
                </tr>
              ))}
              {monthRecords.length===0&&<tr><td colSpan={7} style={{textAlign:"center",color:GRAY,padding:32}}>No sales recorded for this period.</td></tr>}
            </tbody>
          </table>
        )}
      </div>

      {/* Enter Sales Modal */}
      {modal && (
        <div className="overlay" onClick={()=>setModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{width:460}}>
            <div style={{fontFamily:"'Cinzel',serif",fontSize:18,color:GOLD,marginBottom:20}}>Enter Monthly Sales</div>
            <div style={{display:"flex",flexDirection:"column",gap:14}}>
              <div>
                <label className="lbl">Employee</label>
                <select value={form.employeeEmail} onChange={e=>setForm({...form,employeeEmail:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}>
                  <option value="">— Select Employee —</option>
                  {employees.map(e=><option key={e.id} value={e.email||e.id}>{e.name} — {e.department||""} {!deptHasTiers(e.department)?"(No commission)":""}</option>)}
                </select>
              </div>
              <div>
                <label className="lbl">Month</label>
                <input type="month" value={form.month} onChange={e=>setForm({...form,month:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              <div>
                <label className="lbl">Total Sales This Month</label>
                <input type="number" min="0" placeholder="e.g. 25" value={form.salesCount}
                  onChange={e=>setForm({...form,salesCount:e.target.value})}
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>

              {/* Live preview */}
              {form.employeeEmail && form.salesCount!=="" && (()=>{
                const emp = employees.find(e=>(e.email||e.id)===form.employeeEmail);
                const tierList = getTiersFor(emp?.department);
                if (!tierList.length) return (
                  <div style={{background:"rgba(201,76,76,.08)",border:"1px solid rgba(201,76,76,.3)",borderRadius:6,padding:12,fontSize:13,color:"#C94C4C"}}>
                    ⚠ {emp?.department} has no commission system. This won't auto-update salary.
                  </div>
                );
                const calc = calcSalaryFromSales(Number(form.salesCount), tierList);
                return (
                  <div style={{background:"rgba(201,168,76,.08)",border:"1px solid rgba(201,168,76,.3)",borderRadius:8,padding:16}}>
                    <div style={{fontFamily:"'Cinzel',serif",fontSize:12,color:GOLD,letterSpacing:1,marginBottom:12}}>SALARY PREVIEW ({emp?.department})</div>
                    <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8}}>
                      {[["Tier",calc?.tier?.label,GOLD],["Base",PKR(calc?.base),WHITE],["Punctuality",PKR(calc?.punctuality),"#4CC98A"],["Total",PKR(calc?.total),GOLD]].map(([k,v,c])=>(
                        <div key={k} style={{padding:8,background:BLACK3,borderRadius:4}}>
                          <div style={{fontSize:10,color:GRAY,marginBottom:3}}>{k}</div>
                          <div style={{fontSize:14,color:c,fontWeight:700}}>{v}</div>
                        </div>
                      ))}
                    </div>
                    {calc?.belowMin && <div style={{marginTop:10,fontSize:12,color:"#C94C4C",fontWeight:600}}>⚠ Below minimum tier</div>}
                  </div>
                );
              })()}

              <div>
                <label className="lbl">Notes (optional)</label>
                <input value={form.notes} onChange={e=>setForm({...form,notes:e.target.value})} placeholder="Any remarks..."
                  style={{background:BLACK3,border:"1px solid #444",color:WHITE,padding:"10px 14px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:14,width:"100%",outline:"none"}}/>
              </div>
              <div style={{display:"flex",gap:12}}>
                <button className="gold-btn" onClick={saveSales} disabled={saving} style={{flex:1}}>{saving?<Spinner size={14}/>:"Save & Update Payroll"}</button>
                <button className="ghost-btn" onClick={()=>setModal(false)} style={{flex:1}}>Cancel</button>
              </div>
            </div>
          </div>
        </div>
      )}

      {/* Tier Manager Modal — Full CRUD */}
      {tierModal && (
        <div className="overlay" onClick={()=>setTierModal(false)}>
          <div className="modal" onClick={e=>e.stopPropagation()} style={{width:640,maxHeight:"90vh",overflowY:"auto"}}>
            <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:6}}>
              <div style={{fontFamily:"'Cinzel',serif",fontSize:18,color:GOLD}}>Tiers for {tierDept}</div>
              <button className="danger-btn" onClick={removeTierSystemFromDept} style={{fontSize:11}}>Remove System</button>
            </div>
            <div style={{fontSize:12,color:GRAY,marginBottom:18}}>Add, edit, or delete sales tiers for this department</div>

            {/* Existing tiers list */}
            <div style={{marginBottom:20}}>
              <table>
                <thead><tr><th>Label</th><th>Min</th><th>Max</th><th>Base</th><th>Punct.</th><th>Extra/Sale</th><th>Actions</th></tr></thead>
                <tbody>
                  {getTiersFor(tierDept).map(t=>(
                    <tr key={t.id} style={{background:editingTier?.id===t.id?"rgba(201,168,76,.06)":"transparent"}}>
                      <td style={{fontSize:12,color:GOLD,fontWeight:600}}>{t.label}</td>
                      <td style={{fontSize:12}}>{t.minSales}</td>
                      <td style={{fontSize:12}}>{t.maxSales>=999?"∞":t.maxSales}</td>
                      <td style={{fontSize:12}}>{PKR(t.baseSalary)}</td>
                      <td style={{fontSize:12,color:"#4CC98A"}}>{PKR(t.punctuality)}</td>
                      <td style={{fontSize:12,color:GOLD}}>{t.perExtraSale?PKR(t.perExtraSale):"—"}</td>
                      <td>
                        <div style={{display:"flex",gap:4}}>
                          <button className="ghost-btn" onClick={()=>editTier(t)} style={{padding:"3px 8px",fontSize:11}}>✏️</button>
                          <button className="danger-btn" onClick={()=>deleteTier(t.id)} style={{padding:"3px 7px",fontSize:11}}>✕</button>
                        </div>
                      </td>
                    </tr>
                  ))}
                  {getTiersFor(tierDept).length===0&&<tr><td colSpan={7} style={{textAlign:"center",color:GRAY,padding:20}}>No tiers yet — add one below.</td></tr>}
                </tbody>
              </table>
            </div>

            {/* Add/Edit tier form */}
            <div style={{background:BLACK3,borderRadius:8,padding:16,border:`1px solid ${editingTier?GOLD:"#333"}`}}>
              <div style={{fontFamily:"'Cinzel',serif",fontSize:13,color:GOLD,marginBottom:12}}>{editingTier?"Edit Tier":"+ Add New Tier"}</div>
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr 1fr",gap:10,marginBottom:10}}>
                <div>
                  <label className="lbl">Label</label>
                  <input value={tierForm.label} onChange={e=>setTierForm({...tierForm,label:e.target.value})} placeholder="e.g. 12 Sales"
                    style={{background:BLACK2,border:"1px solid #444",color:WHITE,padding:"8px 10px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}/>
                </div>
                <div>
                  <label className="lbl">Min Sales</label>
                  <input type="number" value={tierForm.minSales} onChange={e=>setTierForm({...tierForm,minSales:e.target.value})} placeholder="12"
                    style={{background:BLACK2,border:"1px solid #444",color:WHITE,padding:"8px 10px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}/>
                </div>
                <div>
                  <label className="lbl">Max Sales (blank=∞)</label>
                  <input type="number" value={tierForm.maxSales} onChange={e=>setTierForm({...tierForm,maxSales:e.target.value})} placeholder="14"
                    style={{background:BLACK2,border:"1px solid #444",color:WHITE,padding:"8px 10px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}/>
                </div>
                <div>
                  <label className="lbl">Base Salary (PKR)</label>
                  <input type="number" value={tierForm.baseSalary} onChange={e=>setTierForm({...tierForm,baseSalary:e.target.value})} placeholder="45000"
                    style={{background:BLACK2,border:"1px solid #444",color:WHITE,padding:"8px 10px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}/>
                </div>
                <div>
                  <label className="lbl">Punctuality (PKR)</label>
                  <input type="number" value={tierForm.punctuality} onChange={e=>setTierForm({...tierForm,punctuality:e.target.value})} placeholder="5000"
                    style={{background:BLACK2,border:"1px solid #444",color:WHITE,padding:"8px 10px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}/>
                </div>
                <div>
                  <label className="lbl">Extra/Sale (top tier only)</label>
                  <input type="number" value={tierForm.perExtraSale} onChange={e=>setTierForm({...tierForm,perExtraSale:e.target.value})} placeholder="2000"
                    style={{background:BLACK2,border:"1px solid #444",color:WHITE,padding:"8px 10px",borderRadius:4,fontFamily:"'Rajdhani',sans-serif",fontSize:13,width:"100%",outline:"none"}}/>
                </div>
              </div>
              <div style={{display:"flex",gap:10}}>
                <button className="gold-btn" onClick={addOrUpdateTier} style={{flex:1,padding:"9px"}}>{editingTier?"Update Tier":"+ Add Tier"}</button>
                {editingTier&&<button className="ghost-btn" onClick={()=>{setEditingTier(null);setTierForm({minSales:"",maxSales:"",baseSalary:"",punctuality:"",label:"",perExtraSale:""});}} style={{flex:1,padding:"9px"}}>Cancel Edit</button>}
              </div>
            </div>

            <button className="ghost-btn" onClick={()=>setTierModal(false)} style={{width:"100%",marginTop:16}}>Close</button>
          </div>
        </div>
      )}
    </div>
  );
}


// ============================================================
//  CONFIG — Vici Dial API settings
//  Jab provider se credentials milein, yahan fill karo
// ============================================================
const VICI_CONFIG = {
  apiUrl:    "https://YOUR_VICI_URL/vicidial/vicidial_api.php", // provider dega
  apiUser:   "YOUR_API_USER",      // provider dega
  apiPass:   "YOUR_API_PASS",      // provider dega
  campaign:  "YOUR_CAMPAIGN_ID",   // apna campaign ID
};

// ============================================================
//  MOCK DATA — Jab tak API nahi milti, yeh use hoga
//  API milne ke baad MOCK_MODE = false kar do
// ============================================================
const MOCK_MODE = true;

const ALL_AGENTS = [
  { name: "Abdul Basit",     role: "JR" },
  { name: "Amisha Bhatti",   role: "JR" },
  { name: "Danish Gill",     role: "JR" },
  { name: "Daniyal Gill",    role: "JR" },
  { name: "Faizan Saeed",    role: "JR" },
  { name: "Kamran Khan",     role: "SR" },
  { name: "Kashif Khan",     role: "SR" },
  { name: "Samar Mazhar",    role: "SR" },
  { name: "Shahram Khokhar", role: "SR" },
  { name: "Yashwa Chriss",   role: "SR" },
];

function rand(a, b) { return Math.floor(Math.random() * (b - a + 1)) + a; }
function todayStr() { return new Date().toISOString().slice(0, 10); }
function offsetStr(d) {
  const dt = new Date(); dt.setDate(dt.getDate() - d);
  return dt.toISOString().slice(0, 10);
}

function mockGenerate(isHist) {
  return ALL_AGENTS.map(a => ({
    ...a,
    dials:  isHist ? rand(80, 200) : rand(40, 95),
    trx:    isHist ? rand(2, 30)   : rand(0, 15),
    status: isHist ? "idle" : (Math.random() < .25 ? "live" : Math.random() < .3 ? "dialing" : "idle"),
  }));
}

// ============================================================
//  VICI API FETCH — Real data (MOCK_MODE = false hone par)
// ============================================================
async function fetchFromVici(mode) {
  try {
    const { apiUrl, apiUser, apiPass, campaign } = VICI_CONFIG;

    // Live agent status
    const liveRes = await fetch(
      `${apiUrl}?source=test&user=${apiUser}&pass=${apiPass}&function=agent_status&campaign=${campaign}&format=json`
    );
    const liveData = await liveRes.json();

    // Call logs for dials count
    const logRes = await fetch(
      `${apiUrl}?source=test&user=${apiUser}&pass=${apiPass}&function=call_log&campaign=${campaign}&format=json`
    );
    const logData = await logRes.json();

    // Map Vici data to our agent format
    return ALL_AGENTS.map(agent => {
      const live = liveData?.agents?.find(a => a.name === agent.name);
      const log  = logData?.agents?.find(a => a.name === agent.name);

      const viciStatus = live?.status || "PAUSED";
      let status = "idle";
      if (["INCALL", "CLOSER"].includes(viciStatus))                    status = "live";
      if (["MANUALPREVIEW", "MANUALDIAL", "DIALING"].includes(viciStatus)) status = "dialing";

      return {
        ...agent,
        dials:  parseInt(log?.calls || 0),
        trx:    parseInt(log?.transfers || 0),
        status: mode === "today" ? status : "idle",
      };
    });
  } catch (err) {
    console.error("Vici API error:", err);
    return mockGenerate(mode !== "today");
  }
}

// ============================================================
//  STYLES — inline (no Tailwind dependency)
// ============================================================
const S = {
  wrap:      { background: "#0d0d0d", borderRadius: 12, border: "1px solid #2a2a2a", fontFamily: "'Courier New', monospace", overflow: "hidden", minHeight: "100%", width: "100%" },
  hd:        { background: "#111", borderBottom: "1px solid #1e1e1e", padding: "10px 16px", display: "flex", justifyContent: "space-between", alignItems: "center", flexWrap: "wrap", gap: 6 },
  htitle:    { color: "#fff", fontSize: 13, fontWeight: 700, letterSpacing: ".08em", textTransform: "uppercase" },
  hsub:      { color: "#555", fontSize: 11, marginTop: 2 },
  clk:       { color: "#e65c00", fontSize: 11, fontWeight: 700 },
  badgeLive: { background: "#e65c00", color: "#fff", fontSize: 10, fontWeight: 700, padding: "2px 8px", borderRadius: 4, letterSpacing: ".05em" },
  badgeHist: { background: "#5c35c8", color: "#fff", fontSize: 10, fontWeight: 700, padding: "2px 8px", borderRadius: 4, letterSpacing: ".05em" },
  datebar:   { background: "#0f0f0f", borderBottom: "1px solid #1a1a1a", padding: "7px 16px", display: "flex", alignItems: "center", gap: 8, flexWrap: "wrap" },
  dlabel:    { color: "#555", fontSize: 10, letterSpacing: ".07em", textTransform: "uppercase" },
  dbOn:      { background: "#1e0e00", border: "1px solid #e65c00", color: "#e65c00", fontSize: 10, fontWeight: 700, padding: "3px 9px", borderRadius: 4, cursor: "pointer", fontFamily: "inherit" },
  dbOff:     { background: "#1a1a1a", border: "1px solid #2a2a2a", color: "#888", fontSize: 10, padding: "3px 9px", borderRadius: 4, cursor: "pointer", fontFamily: "inherit" },
  tabs:      { background: "#0f0f0f", borderBottom: "1px solid #1e1e1e", padding: "0 16px", display: "flex", gap: 6 },
  tabOn:     { padding: "9px 14px 8px", fontSize: 11, fontWeight: 700, letterSpacing: ".07em", textTransform: "uppercase", color: "#e65c00", borderBottom: "2px solid #e65c00", background: "none", border: "none", borderBottom: "2px solid #e65c00", cursor: "pointer", display: "flex", alignItems: "center", gap: 6, fontFamily: "inherit" },
  tabOff:    { padding: "9px 14px 8px", fontSize: 11, fontWeight: 700, letterSpacing: ".07em", textTransform: "uppercase", color: "#444", borderBottom: "2px solid transparent", background: "none", border: "none", borderBottom: "2px solid transparent", cursor: "pointer", display: "flex", alignItems: "center", gap: 6, fontFamily: "inherit" },
  cntOn:     { fontSize: 10, fontWeight: 700, padding: "1px 6px", borderRadius: 10, background: "#2a1000", color: "#e65c00" },
  cntOff:    { fontSize: 10, fontWeight: 700, padding: "1px 6px", borderRadius: 10, background: "#1e1e1e", color: "#555" },
  stats:     { background: "#111", display: "flex", borderBottom: "1px solid #1a1a1a", flexWrap: "wrap" },
  sb:        { flex: 1, minWidth: 110, padding: "10px 14px", borderRight: "1px solid #1a1a1a" },
  sl:        { fontSize: 10, color: "#555", letterSpacing: ".08em", textTransform: "uppercase", marginBottom: 3 },
  leg:       { display: "flex", alignItems: "center", gap: 14, padding: "5px 16px", background: "#0f0f0f", borderBottom: "1px solid #1a1a1a", flexWrap: "wrap" },
  hbanner:   { background: "#0d0820", borderBottom: "1px solid #2a1a5a", padding: "5px 16px", display: "flex", alignItems: "center", gap: 8 },
  ft:        { background: "#111", padding: "7px 16px", display: "flex", justifyContent: "space-between", alignItems: "center", borderTop: "1px solid #1a1a1a", flexWrap: "wrap", gap: 4 },
};

// ============================================================
//  MAIN COMPONENT
// ============================================================
function LeaderboardPage() {
  const [allData, setAllData]       = useState([]);
  const [sortKey, setSortKey]       = useState("dials");
  const [activeTab, setActiveTab]   = useState("all");
  const [mode, setMode]             = useState("today");
  const [customDate, setCustomDate] = useState("");
  const [clock, setClock]           = useState("");
  const [lastRefresh, setLastRefresh] = useState("—");

  const isLive = mode === "today" || (mode === "custom" && customDate === todayStr());

  const dateLabel = () => {
    if (mode === "today")     return todayStr() + " (Today)";
    if (mode === "yesterday") return offsetStr(1) + " (Yesterday)";
    if (mode === "7d")        return offsetStr(6) + " → " + todayStr();
    return customDate;
  };

  const loadData = useCallback(async () => {
    const data = MOCK_MODE ? mockGenerate(!isLive) : await fetchFromVici(mode);
    setAllData(data);
    setLastRefresh(new Date().toLocaleTimeString());
  }, [mode, isLive]);

  useEffect(() => { loadData(); }, [loadData]);

  useEffect(() => {
    if (!isLive) return;
    const t = setInterval(loadData, 10000);
    return () => clearInterval(t);
  }, [isLive, loadData]);

  useEffect(() => {
    const t = setInterval(() => setClock(new Date().toLocaleTimeString("en-PK", { hour12: false })), 1000);
    return () => clearInterval(t);
  }, []);

  const tabAgents = allData.filter(a =>
    activeTab === "sr" ? a.role === "SR" :
    activeTab === "jr" ? a.role === "JR" : true
  );

  const sorted = [...tabAgents].sort((a, b) => {
    if (sortKey === "status") { const o = { live: 0, dialing: 1, idle: 2 }; return o[a.status] - o[b.status]; }
    return b[sortKey] - a[sortKey];
  });

  const totalDials   = tabAgents.reduce((s, a) => s + a.dials, 0);
  const liveCnt      = tabAgents.filter(a => a.status === "live").length;
  const dialingCnt   = tabAgents.filter(a => a.status === "dialing").length;
  const totalTrx     = tabAgents.reduce((s, a) => s + a.trx, 0);
  const maxD         = Math.max(...sorted.map(a => a.dials), 1);
  const maxT         = Math.max(...sorted.map(a => a.trx), 1);

  const tabName = activeTab === "all" ? "ALL QUALIFIERS" : activeTab === "sr" ? "SR FALCO" : "JR FALCO";
  const rankColors = ["#ffd700", "#c0c0c0", "#cd7f32"];

  const handlePreset = (p) => {
    setMode(p); setCustomDate("");
  };

  const handleCustom = (v) => {
    if (!v) return;
    setCustomDate(v);
    setMode("custom");
  };

  return (
    <div style={S.wrap}>

      {/* HEADER */}
      <div style={S.hd}>
        <div>
          <div style={S.htitle}>Vici Dial — Agent Leaderboard</div>
          <div style={S.hsub}>DATE: {dateLabel()} &nbsp;|&nbsp; {tabName}</div>
        </div>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <span style={S.clk}>{clock}</span>
          <span style={isLive ? S.badgeLive : S.badgeHist}>{isLive ? "LIVE" : "HISTORY"}</span>
        </div>
      </div>

      {/* DATE BAR */}
      <div style={S.datebar}>
        <span style={S.dlabel}>Date:</span>
        {[["today","Today"],["yesterday","Yesterday"],["7d","Last 7 Days"]].map(([key,label])=>(
          <button key={key} style={mode===key ? S.dbOn : S.dbOff} onClick={()=>handlePreset(key)}>{label}</button>
        ))}
        <span style={{ color:"#2a2a2a" }}>|</span>
        <span style={S.dlabel}>Custom:</span>
        <input
          type="date" max={todayStr()}
          value={customDate}
          onChange={e => handleCustom(e.target.value)}
          style={{ background:"#1a1a1a", border:"1px solid #2a2a2a", color:"#ccc", fontSize:10, padding:"3px 7px", borderRadius:4, fontFamily:"inherit" }}
        />
      </div>

      {/* HISTORY BANNER */}
      {!isLive && (
        <div style={S.hbanner}>
          <span style={{ color:"#9b6dff", fontSize:13 }}>📅</span>
          <span style={{ color:"#9b6dff", fontSize:11 }}>Historical — {dateLabel()}</span>
        </div>
      )}

      {/* TABS */}
      <div style={S.tabs}>
        {[["all","All Qualifiers"],["sr","SR Falco"],["jr","JR Falco"]].map(([key,label])=>(
          <button key={key} style={activeTab===key ? S.tabOn : S.tabOff} onClick={()=>setActiveTab(key)}>
            {label}
            <span style={activeTab===key ? S.cntOn : S.cntOff}>
              {key==="all" ? allData.length : allData.filter(a=>a.role===key.toUpperCase()).length}
            </span>
          </button>
        ))}
      </div>

      {/* STATS */}
      <div style={S.stats}>
        {[
          { label:"Manual Dials",    val: totalDials,            color:"#e65c00", sub: isLive?"Shift total":"Day total" },
          { label:"Live Calls",      val: isLive ? liveCnt:"—",  color:"#00c853", sub: isLive?"On call now":"Historical" },
          { label:"Manual Dialing",  val: isLive ? dialingCnt:"—", color:"#ff9100", sub: isLive?"Dialing now":"Historical" },
          { label:"Total TRX",       val: totalTrx,              color:"#2979ff", sub: isLive?"Transactions":"Day total" },
        ].map((s,i)=>(
          <div key={i} style={{ ...S.sb, borderRight: i<3?"1px solid #1a1a1a":"none" }}>
            <div style={S.sl}>{s.label}</div>
            <div style={{ fontSize:22, fontWeight:700, color:s.color }}>{s.val}</div>
            <div style={{ fontSize:10, color:"#444", marginTop:1 }}>{s.sub}</div>
          </div>
        ))}
      </div>

      {/* LEGEND */}
      <div style={S.leg}>
        {[["#00c853","Live Call"],["#ff9100","Manual Dialing"],["#2a2a2a","Idle"]].map(([c,l])=>(
          <div key={l} style={{ display:"flex", alignItems:"center", gap:5, fontSize:10, color:"#555" }}>
            <div style={{ width:7, height:7, borderRadius:"50%", background:c }}/>
            {l}
          </div>
        ))}
        <div style={{ display:"flex", alignItems:"center", gap:5, fontSize:10, color:"#555" }}>
          <span style={{ fontSize:9, fontWeight:700, padding:"1px 5px", borderRadius:3, background:"#1a1030", color:"#9b6dff" }}>SR</span>Senior
        </div>
        <div style={{ display:"flex", alignItems:"center", gap:5, fontSize:10, color:"#555" }}>
          <span style={{ fontSize:9, fontWeight:700, padding:"1px 5px", borderRadius:3, background:"#1e0e00", color:"#e65c00" }}>JR</span>Junior
        </div>
      </div>

      {/* TABLE */}
      <div style={{ overflowX:"auto" }}>
        <table style={{ width:"100%", borderCollapse:"collapse", fontSize:12 }}>
          <thead>
            <tr style={{ background:"#111", borderBottom:"2px solid #1e1e1e" }}>
              {[["#",""],["Agent",""],["Dials","dials"],["TRX","trx"],["Status","status"]].map(([label,key])=>(
                <th key={label}
                  onClick={()=>key&&setSortKey(key)}
                  style={{ padding:"8px 10px", textAlign:"left", fontSize:10, letterSpacing:".07em", textTransform:"uppercase", fontWeight:700, cursor:key?"pointer":"default", whiteSpace:"nowrap", color: sortKey===key?"#e65c00":"#444" }}
                >{label}{key?" ▼":""}</th>
              ))}
            </tr>
          </thead>
          <tbody>
            {sorted.map((a, i) => {
              const isLiveRow     = a.status === "live";
              const isDialingRow  = a.status === "dialing";
              const rowBg = isLiveRow ? "#071407" : isDialingRow ? "#1a0e00" : "transparent";
              const dp = Math.round(a.dials / maxD * 100);
              const tp = Math.round(a.trx / maxT * 100);
              return (
                <tr key={a.name} style={{ borderBottom:"1px solid #161616", background:rowBg }}>
                  <td style={{ padding:"9px 10px", color: rankColors[i]||"#444", fontWeight:700, fontSize:13, width:28 }}>{i+1}</td>
                  <td style={{ padding:"9px 10px", color:"#fff", fontWeight:600 }}>
                    {a.name}
                    <span style={{ fontSize:9, fontWeight:700, padding:"1px 5px", borderRadius:3, marginLeft:4, background:a.role==="SR"?"#1a1030":"#1e0e00", color:a.role==="SR"?"#9b6dff":"#e65c00" }}>{a.role}</span>
                  </td>
                  <td style={{ padding:"9px 10px" }}>
                    <span style={{ color:"#e65c00", fontWeight:700, fontSize:13 }}>{a.dials}</span>
                    <div style={{ background:"#1a1a1a", borderRadius:2, height:3, width:56, marginTop:3 }}>
                      <div style={{ height:3, borderRadius:2, background:"#e65c00", width:`${dp}%` }}/>
                    </div>
                  </td>
                  <td style={{ padding:"9px 10px" }}>
                    <span style={{ color:"#2979ff", fontWeight:700, fontSize:13 }}>{a.trx}</span>
                    <div style={{ background:"#1a1a1a", borderRadius:2, height:3, width:56, marginTop:3 }}>
                      <div style={{ height:3, borderRadius:2, background:"#2979ff", width:`${tp}%` }}/>
                    </div>
                  </td>
                  <td style={{ padding:"9px 10px" }}>
                    {!isLive ? <span style={{ color:"#2a2a2a" }}>—</span>
                    : isLiveRow    ? <span style={{ display:"inline-flex", alignItems:"center", gap:4, fontSize:11, fontWeight:700, color:"#00c853" }}><span style={{ width:7, height:7, borderRadius:"50%", background:"#00c853", display:"inline-block", animation:"pulse 1.2s infinite" }}/>Live Call</span>
                    : isDialingRow ? <span style={{ display:"inline-flex", alignItems:"center", gap:4, fontSize:11, fontWeight:700, color:"#ff9100" }}><span style={{ width:7, height:7, borderRadius:"50%", background:"#ff9100", display:"inline-block", animation:"pulse 1.2s infinite" }}/>Dialing</span>
                    : <span style={{ color:"#2a2a2a", fontSize:11 }}>Idle</span>}
                  </td>
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>

      {/* FOOTER */}
      <div style={S.ft}>
        <span style={{ fontSize:10, color:"#333" }}>Last refresh: {lastRefresh}</span>
        <span style={{ fontSize:10, color:"#333" }}>{isLive ? "Auto-refresh: 10s" : "No auto-refresh"}</span>
        <button onClick={loadData} style={{ background:"none", border:"1px solid #2a2a2a", color:"#555", fontSize:10, padding:"3px 10px", borderRadius:4, cursor:"pointer", fontFamily:"inherit" }}>⟳ Refresh</button>
      </div>

      <style>{`@keyframes pulse{0%,100%{opacity:1;transform:scale(1)}50%{opacity:.4;transform:scale(.8)}}`}</style>
    </div>
  );
}


// ─── PLACEHOLDER ─────────────────────────────────────────────────────────────
const PlaceholderPage = ({title,icon,desc}) => (
  <div className="fade" style={{ display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", minHeight:400 }}>
    <div style={{ fontSize:56, marginBottom:16, opacity:.4 }}>{icon}</div>
    <div style={{ fontFamily:"'Cinzel',serif", fontSize:22, color:WHITE, marginBottom:8 }}>{title}</div>
    <div style={{ fontSize:14, color:GRAY, marginBottom:24 }}>{desc}</div>
    <div style={{ padding:"10px 20px", background:"rgba(201,168,76,.08)", border:"1px solid rgba(201,168,76,.2)", borderRadius:6, fontSize:13, color:GOLD }}>🚧 Ask to build this module next!</div>
  </div>
);

// ─── ROOT ─────────────────────────────────────────────────────────────────────
// ─── CLIENT-SIDE AUTO ATTENDANCE CHECK ────────────────────────────────────────
async function runAutoAttendanceCheck() {
  try {
    const now = new Date();
    const pktNow = new Date(now.getTime() + 5 * 60 * 60 * 1000);
    const pktHour = pktNow.getHours();
    const pktMinute = pktNow.getMinutes();
    const todayStr = pktNow.toISOString().slice(0,10);

    // GUARD: If already ran NCNS marking today, skip (prevent duplicate runs)
    const ncnsRanKey = `ncnsRan_${todayStr}`;
    const alreadyMarked = localStorage.getItem(ncnsRanKey);
    if (alreadyMarked && pktHour >= 20) {
      console.log("NCNS already marked today — skipping");
      return;
    }

    // SAFETY: Never run NCNS check before 8:00 PM PKT
    // Shift starts at 6PM, grace period until 8PM before marking absent
    if (pktHour < 20) {
      console.log(`Auto check skipped — only runs after 8PM PKT. Current PKT hour: ${pktHour}`);
      // CLEANUP: Remove any wrongly auto-marked NCNS/absent for today if before 8PM
      const wrongSnap = await getDocs(collection(db,"attendance"));
      const wrongRecs = wrongSnap.docs.filter(d=>{
        const r = d.data();
        return r.date===todayStr && r.autoMarked===true && (r.status==="ncns"||r.status==="absent"||r.status==="on_leave");
      });
      for (const d of wrongRecs) {
        await deleteDoc(doc(db,"attendance",d.id));
        console.log(`Removed wrong auto-mark: ${d.data().employeeName}`);
      }
      // Also clean wrong notifications
      const notifSnap = await getDocs(collection(db,"notifications"));
      const wrongNotifs = notifSnap.docs.filter(d=>{
        const n = d.data();
        return n.date===todayStr && n.type==="ncns";
      });
      for (const d of wrongNotifs) {
        await deleteDoc(doc(db,"notifications",d.id));
      }
      return;
    }

    const todayStr2 = todayStr; // same variable, just clarity
    const yesterday = new Date(pktNow);
    yesterday.setDate(yesterday.getDate()-1);
    const yesterdayStr = yesterday.toISOString().slice(0,10);
    const isWeekend = ds => { const d = new Date(ds+"T00:00:00").getDay(); return d===0||d===6; };
    const fineDoc = await getDoc(doc(db,"settings","fines"));
    const NCNS_FINE = fineDoc.exists() ? (fineDoc.data().nncsFine||2000) : 2000;
    const [empSnap, leavesSnap, attSnap] = await Promise.all([
      getDocs(collection(db,"employees")),
      getDocs(collection(db,"leaves")),
      getDocs(collection(db,"attendance")),
    ]);
    const employees = empSnap.docs.map(d=>({...d.data(),id:d.id}));
    const allLeaves = leavesSnap.docs.map(d=>d.data());
    const allAtt    = attSnap.docs.map(d=>d.data());
    if (pktHour >= 20 && !isWeekend(todayStr)) {
      for (const emp of employees) {
        const email = emp.email||emp.id;
        if (!email) continue;
        if (allAtt.find(r=>r.employeeEmail===email&&r.date===todayStr)) continue;
        const attId = `${email}_${todayStr}`;
        const approved = allLeaves.find(l=>l.employeeEmail===email&&l.status==="approved"&&l.fromDate<=todayStr&&l.toDate>=todayStr);
        if (approved) {
          await setDoc(doc(db,"attendance",attId),{ employeeEmail:email, employeeName:emp.name, date:todayStr, checkIn:null, checkOut:null, status:"on_leave", leaveType:approved.leaveType, fine:0, fineReason:"", notes:`On approved ${approved.leaveType} leave`, autoMarked:true, createdAt:serverTimestamp() });
          continue;
        }
        const ncnsCount = allAtt.filter(r=>r.employeeEmail===email&&r.status==="ncns").length+1;
        const warningNote = ncnsCount===1?"1st NCNS — Fine applied":ncnsCount===2?"2nd NCNS — Written Warning issued":"3rd NCNS — TERMINATION";
        const rejected = allLeaves.find(l=>l.employeeEmail===email&&l.status==="rejected"&&l.fromDate<=todayStr&&l.toDate>=todayStr);
        const reason = rejected?`NCNS — Leave rejected. ${warningNote}`:`NCNS — No call, no show. ${warningNote}`;
        await setDoc(doc(db,"attendance",attId),{ employeeEmail:email, employeeName:emp.name, date:todayStr, checkIn:null, checkOut:null, status:"ncns", fine:NCNS_FINE, fineReason:reason, ncnsCount, warningNote, autoMarked:true, createdAt:serverTimestamp() });
        await setDoc(doc(db,"notifications",`ncns_${email}_${todayStr}`),{ type:"ncns", employeeEmail:email, employeeName:emp.name, date:todayStr, ncnsCount, warningNote, fine:NCNS_FINE, message:`🚨 NCNS: ${emp.name} — ${warningNote} on ${todayStr}`, createdAt:serverTimestamp(), read:false });
      }
    }
    if (pktHour >= 5 && !isWeekend(yesterdayStr)) {
      const noCheckout = attSnap.docs.filter(d=>{const r=d.data(); return r.date===yesterdayStr&&r.checkIn&&!r.checkOut&&!["absent","ncns"].includes(r.status);});
      for (const d of noCheckout) {
        await updateDoc(doc(db,"attendance",d.id),{ status:"absent", notes:"Auto-absent: no checkout by 5AM PKT", autoAbsentNoCheckout:true });
      }
    }
    console.log("✅ Auto attendance check complete");
  } catch(e) { console.log("Auto check skipped:", e.message); }
}

export default function App() {
  const [authUser, setAuthUser] = useState(undefined);
  const [users,    setUsers]    = useState([]);
  const [depts,    setDepts]    = useState([]);
  const [page,     setPage]     = useState("dashboard");

  useEffect(() => {
    seedSuperAdmin();
    runAutoAttendanceCheck();
    const unsub = onAuthStateChanged(auth, async fbUser => {
      if (!fbUser) { setAuthUser(null); return; }
      try {
        const snap = await getDoc(doc(db,"users",fbUser.email));
        if (snap.exists()) {
          const data = snap.data();
          const normalized = {
            ...data,
            uid: fbUser.uid,
            email: fbUser.email,
            name: data.name || data.NAME || data.Name || "",
            role: (data.role || data.Role || data.ROLE || "employee").toLowerCase().trim(),
            active: data.active !== undefined ? data.active : true,
            department: data.department || data.Department || "",
          };
          if (normalized.active) setAuthUser(normalized);
          else { await signOut(auth); setAuthUser(null); }
        } else { await signOut(auth); setAuthUser(null); }
      } catch(e) { setAuthUser(null); }
    });
    return unsub;
  }, []);

  useEffect(() => {
    if (!authUser || authUser.role!=="super_admin") return;
    const unsub = onSnapshot(collection(db,"users"), snap => {
      setUsers(snap.docs.map(d => {
        const data = d.data();
        return {
          ...data,
          email: d.id,
          name: data.name || data.NAME || data.Name || "",
          role: data.role || data.Role || data.ROLE || "employee",
          active: data.active !== undefined ? data.active : (data.Status === "Active"),
          department: data.department || data.Department || data.DEPARTMENT || "",
        };
      }));
    });
    return unsub;
  }, [authUser]);

  useEffect(() => {
    if (!authUser) return;
    const unsub = onSnapshot(collection(db,"departments"), snap => {
      setDepts(snap.docs.map(d => ({ ...d.data(), id:d.id })));
    });
    return unsub;
  }, [authUser]);

  const logout = async () => { await signOut(auth); setAuthUser(null); setPage("dashboard"); };

  if (authUser===undefined) return (
    <>
      <GlobalStyle/>
      <div style={{ minHeight:"100vh", display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", background:BLACK, gap:20 }}>
        <Logo big/><Spinner size={32}/>
        <div style={{ color:GRAY, fontSize:13 }}>Connecting to Firebase...</div>
      </div>
    </>
  );

  if (!authUser) return <><GlobalStyle/><LoginPage onLogin={setAuthUser}/></>;

  const renderPage = () => {
    switch(page) {
      case "dashboard":   return <Dashboard user={authUser} users={users} depts={depts}/>;
      // Users merged into Employees page
      case "departments": return <DeptsPage depts={depts} setDepts={setDepts}/>;
      case "employees":   return <EmployeesPage depts={depts} currentUser={authUser}/>;
      case "attendance":  return <AttendancePage user={authUser}/>;
      case "payroll":     return <PayrollPage user={authUser}/>;
      case "leaves":      return <LeavePage user={authUser}/>;
      case "sales":       return <SalesPage user={authUser}/>;
      case "leaderboard": return <div style={{width:"100%", minHeight:"100vh", background:"#0d0d0d"}}><LeaderboardPage/></div>;
      case "settings":    return <SettingsPage user={authUser}/>;
      default: return null;
    }
  };

  return (
    <>
      <GlobalStyle/>
      <div style={{ display:"flex", minHeight:"100vh", background:BLACK }}>
        <Sidebar user={authUser} page={page} setPage={setPage} onLogout={logout}/>
        <main style={{ marginLeft:230, flex:1, padding:"0", minHeight:"100vh", background:"#0D0D0D" }}>
          <div style={{ padding:"20px 36px 0 36px", position:"sticky", top:0, background:BLACK, zIndex:50 }}>
          <TopBar user={authUser} onLogout={logout} page={page} />
          </div>
          <div style={{ padding:"24px 36px" }}>
          {renderPage()}
          </div>
        </main>
      </div>
    </>
  );
}
