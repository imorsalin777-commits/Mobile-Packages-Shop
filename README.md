
<!DOCTYPE html>
<html lang="bn">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Mobile Package Shop</title>
<style>
*{box-sizing:border-box}
body{margin:0;font-family:Arial,'Noto Sans Bengali',sans-serif;background:#f4f7fb;color:#172033}
header{background:linear-gradient(135deg,#087e68,#095c9d);color:white;padding:24px 16px;text-align:center}
header h1{margin:0 0 8px}
header p{margin:0}
.wrap{max-width:900px;margin:auto;padding:16px}
.filters,.card{background:white;border-radius:14px;padding:16px;margin-bottom:14px;box-shadow:0 2px 12px #0000000c}
.filters{display:flex;flex-wrap:wrap;gap:8px}
.filters button{border:0;border-radius:24px;padding:10px 14px;background:#e8eef5;cursor:pointer}
.filters button.active{background:#087e68;color:white}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(230px,1fr));gap:14px}
.card h3{margin:4px 0 8px}
.price{font-size:22px;font-weight:bold;color:#087e68}
.card button,.submit{background:#087e68;color:white;border:0;padding:11px 16px;border-radius:9px;cursor:pointer;width:100%;margin-top:10px}
.muted{color:#64748b;font-size:14px}
.formrow{display:grid;gap:10px}
input,select{padding:12px;border:1px solid #ccd5df;border-radius:9px;width:100%}
#status{white-space:pre-wrap;padding:10px}
.footer{text-align:center;color:#6b7280;padding:22px;font-size:13px}
</style>
</head>
<body>
<header>
<h1>📱 Mobile Package Shop</h1>
<p>আপনার পছন্দের মোবাইল প্যাকেজ নির্বাচন করুন</p>
</header>

<div class="wrap">
<section class="filters" id="filters"></section>
<div id="catalog" class="grid"></div>

<section class="card">
<h2>অর্ডার করুন</h2>
<p class="muted" id="selected">কোনো প্যাকেজ নির্বাচন করা হয়নি।</p>
<div class="formrow">
<input id="customer" placeholder="আপনার নাম">
<input id="phone" placeholder="মোবাইল নম্বর" inputmode="tel">
<button class="submit" id="send">অর্ডার জমা দিন</button>
</div>
<div id="status"></div>
</section>

<section class="card">
<h3>অ্যাডমিন লগইন (ঐচ্ছিক)</h3>
<div class="formrow">
<input id="email" type="email" placeholder="Admin email">
<input id="password" type="password" placeholder="Password">
<button class="submit" id="login">Admin Login</button>
<button class="submit" id="logout">Logout</button>
</div>
<div id="adminstatus"></div>
</section>

<div class="footer">Mobile Package Shop · Demo website</div>
</div>

<script type="module">
import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js';
import { getFirestore, collection, getDocs, addDoc, serverTimestamp } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js';
import { getAuth, signInWithEmailAndPassword, signOut } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js';

const firebaseConfig={
apiKey:'AIzaSyBEKZR-_Uw7GIAdTqn2KxfQ3mt93O8tCY4',
authDomain:'mobile-package-shop.firebaseapp.com',
projectId:'mobile-package-shop',
storageBucket:'mobile-package-shop.firebasestorage.app',
messagingSenderId:'298678365618',
appId:'1:298678365618:web:b2ba3f86030ae1c39948ab',
measurementId:'G-M9JKQ9XLQ9'
};

const app=initializeApp(firebaseConfig);
const db=getFirestore(app);
const auth=getAuth(app);

const operators=['সব','Grameenphone','Banglalink','Robi','Airtel','Teletalk'];
let active='সব',selected=null,packages=[];

const filters=document.getElementById('filters');
const catalog=document.getElementById('catalog');

operators.forEach(o=>{
let b=document.createElement('button');
b.textContent=o;
b.classList.toggle('active',o===active);
b.onclick=()=>{
active=o;
filters.querySelectorAll('button').forEach(x=>x.classList.toggle('active',x===b));
render();
};
filters.append(b);
});

function render(){
catalog.innerHTML='';
let list=packages.filter(p=>(active==='সব'||p.operator===active)&&p.active!==false);

if(!list.length){
catalog.innerHTML='<div class="card">এই অপারেটরে বর্তমানে কোনো প্যাকেজ নেই।</div>';
return;
}

list.forEach(p=>{
let c=document.createElement('article');
c.className='card';
c.innerHTML=`<h3>${safe(p.name||'Package')}</h3>
<div>${safe(p.operator||'')} · ${safe(p.type||'')}</div>
<div class="price">৳${Number(p.price||0)}</div>
<button>নির্বাচন করুন</button>`;

c.querySelector('button').onclick=()=>{
selected=p;
document.getElementById('selected').textContent=
`${p.name} · ${p.operator} · ৳${p.price}`;
};
catalog.append(c);
});
}

function safe(v){
return String(v).replace(/[&<>"']/g,c=>({
'&':'&amp;',
'<':'&lt;',
'>':'&gt;',
'"':'&quot;',
"'":'&#39;'
}[c]));
}

async function load(){
try{
let snap=await getDocs(collection(db,'packages'));
packages=snap.docs.map(d=>({id:d.id,...d.data()}));
render();
}catch(e){
catalog.innerHTML='<div class="card">ডেটাবেস থেকে প্যাকেজ আনতে সমস্যা হয়েছে। Firestore Rules যাচাই করুন।</div>';
console.error(e);
}
}

document.getElementById('send').onclick=async()=>{
let name=document.getElementById('customer').value.trim();
let phone=document.getElementById('phone').value.trim();
let status=document.getElementById('status');

if(!selected){
status.textContent='আগে একটি প্যাকেজ নির্বাচন করুন।';
return;
}

if(!name||!phone){
status.textContent='নাম ও ফোন নম্বর দিন।';
return;
}

status.textContent='অর্ডার জমা হচ্ছে…';

try{
await addDoc(collection(db,'orders'),{
customerName:name,
phone,
packageName:selected.name,
operator:selected.operator,
type:selected.type||'',
price:Number(selected.price||0),
status:'pending',
createdAt:serverTimestamp()
});

status.textContent='অর্ডার জমা হয়েছে! (এটি এখনো রিচার্জ নয়; অ্যাডমিন যাচাই করবেন।)';
selected=null;
document.getElementById('selected').textContent='কোনো প্যাকেজ নির্বাচন করা হয়নি।';

}catch(e){
status.textContent='অর্ডার জমা হয়নি। Rules ঠিক করতে হবে।';
console.error(e);
}
};

document.getElementById('login').onclick=async()=>{
try{
await signInWithEmailAndPassword(
auth,
document.getElementById('email').value,
document.getElementById('password').value
);
document.getElementById('adminstatus').textContent='Admin login সফল হয়েছে (এই ডেমোতে প্যানেল সীমিত)।';
}catch(e){
document.getElementById('adminstatus').textContent='Login ব্যর্থ: email/password যাচাই করুন.';
}
};

document.getElementById('logout').onclick=async()=>{
await signOut(auth);
document.getElementById('adminstatus').textContent='Logout হয়েছে.';
};

load();
</script>
</body>
</html>
