
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#101010">
<title>Shift Tip Calculator</title>
<style>
* { box-sizing: border-box; }
body {
  margin: 0; padding: 20px; background: #101010; color: #fff;
  font-family: -apple-system, BlinkMacSystemFont, sans-serif;
}
main { max-width: 650px; margin: auto; }
h1 { font-size: 27px; margin-bottom: 5px; }
.sub { color: #aaa; margin-bottom: 24px; }
.card {
  background: #1c1c1e; padding: 18px;
  border-radius: 16px; margin: 14px 0;
}
label { display: block; color: #ccc; font-size: 14px; margin-bottom: 7px; }
input {
  display: block; width: 100%; padding: 12px;
  border-radius: 9px; border: 1px solid #444;
  background: #29292b; color: white; font-size: 16px;
  margin-bottom: 13px;
}
button {
  padding: 13px 16px; border: 0; border-radius: 10px;
  background: #ff9f0a; color: #171717; font-size: 15px;
  font-weight: 650; cursor: pointer; margin: 3px 3px 3px 0;
}
button.secondary { background: #39393d; color: white; }
button.delete { background: #512626; color: white; }
.row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.stat { display: flex; justify-content: space-between; gap: 12px; margin: 12px 0; }
.stat strong { text-align: right; }
.big { font-size: 26px; font-weight: 700; }
.host {
  border: 1px solid #414145; border-radius: 12px;
  padding: 14px; margin: 12px 0;
}
.host h3 { margin: 0 0 12px; }
.result { color: #65d6a1; font-size: 21px; font-weight: 700; }
.note { color: #aaa; font-size: 13px; line-height: 1.5; }
hr { border: 0; border-top: 1px solid #414145; margin: 17px 0; }
#report { display: none; }
@media print {
  body { background: white !important; color: black !important; padding: 0; }
  main > *:not(#report) { display: none !important; }
  #report { display: block !important; }
  #report table { width: 100%; border-collapse: collapse; }
  #report th, #report td {
    border: 1px solid #bbb; padding: 8px; text-align: left;
  }
  #report th { background: #eee !important; }
  #report h1, #report h2 { color: black; }
  #report .report-total { font-size: 20px; font-weight: bold; }
  #report p { line-height: 1.5; }
}
</style>
</head>
<body>
<main>
  <h1>🍣 Shift Tip Calculator</h1>
  <div class="sub">Calculate tips and save shift records.</div>

  <section class="card">
    <h2>Shift details</h2>
    <label for="shiftDate">Shift date</label>
    <input id="shiftDate" type="date">

    <label for="shiftName">Shift name or notes (optional)</label>
    <input id="shiftName" placeholder="e.g. Friday dinner">

    <label for="totalTips">Total tips collected ($)</label>
    <input id="totalTips" type="number" min="0" step="0.01"
      value="1000" oninput="calculate()">

    <div class="stat">
      <span>Sushi bar — 40%</span>
      <strong id="barAmount">$400.00</strong>
    </div>
    <div class="stat">
      <span>Host pool — 60%</span>
      <strong id="hostAmount">$600.00</strong>
    </div>
    <p class="note">The sushi bar receives a fixed 40%. The remaining 60% is split between hosts based on their weighted hours.</p>
  </section>

  <section class="card">
    <h2>Host employees</h2>
    <p class="note">Set each employee's rate individually. For example, 100% is the standard rate and 70% is a reduced rate.</p>
    <div id="hosts"></div>
    <button onclick="addHost()">+ Add host</button>
    <button class="secondary" onclick="saveStaff()">Save employee list</button>
    <button class="secondary" onclick="loadStaff()">Load saved employees</button>
  </section>

  <section class="card">
    <h2>Shift results</h2>
    <div class="stat"><span>Total host hours</span><strong id="totalHours">0</strong></div>
    <div class="stat"><span>Total weighted hours</span><strong id="weightedHours">0</strong></div>
    <hr>
    <div id="results"></div>
    <hr>
    <div class="stat">
      <span>Total distributed</span><strong class="big" id="grandTotal">$0.00</strong>
    </div>
    <button onclick="downloadReport()">Download / Print PDF report</button>
    <button class="secondary" onclick="saveRecord()">Save shift record</button>
    <button class="secondary" onclick="showRecords()">View saved shifts</button>
    <p class="note">To save a PDF on iPhone: tap Download / Print PDF report, tap Share or Print in the print screen, then save the PDF to Files.</p>
  </section>

  <section class="card" id="recordsCard" hidden>
    <h2>Saved shift records</h2>
    <div id="records"></div>
    <button class="delete" onclick="clearRecords()">Delete all saved shifts</button>
  </section>

  <section id="report"></section>
</main>

<script>
const $ = id => document.getElementById(id);
const money = n => (Number(n)||0).toLocaleString("en-US", {
  style:"currency", currency:"USD"
});
let nextId = 0;

function addHost(name="", rate="100", hours="") {
  const id = ++nextId;
  const div = document.createElement("div");
  div.className = "host";
  div.dataset.id = id;
  div.innerHTML = `
    <h3>Host</h3>
    <label>Employee name</label>
    <input class="name" placeholder="Employee name">
    <div class="row">
      <div><label>Hours worked</label>
      <input class="hours" type="number" min="0" step="0.25" placeholder="8"></div>
      <div><label>Tip rate (%)</label>
      <input class="rate" type="number" min="0" step="1" placeholder="100"></div>
    </div>
    <div class="result payout">Payout: $0.00</div>
    <button class="delete remove">Remove employee</button>`;
  div.querySelector(".name").value = name;
  div.querySelector(".hours").value = hours;
  div.querySelector(".rate").value = rate;
  div.querySelectorAll("input").forEach(el =>
    el.addEventListener("input", calculate));
  div.querySelector(".remove").onclick = () => {
    div.remove(); calculate();
  };
  $("hosts").appendChild(div);
  calculate();
}

function getHosts() {
  return [...document.querySelectorAll(".host")].map(el => ({
    id: el.dataset.id,
    name: el.querySelector(".name").value.trim() || "Unnamed host",
    hours: Math.max(0, Number(el.querySelector(".hours").value)||0),
    rate: Math.max(0, Number(el.querySelector(".rate").value)||0),
    el
  }));
}

function calculate() {
  const raw = $("totalTips").value;
  const total = Math.max(0, Number(raw)||0);
  const bar = total * 0.40;
  const pool = total * 0.60;
  const hosts = getHosts();
  const sumHours = hosts.reduce((s,h)=>s+h.hours,0);
  const weights = hosts.map(h=>h.hours*h.rate/100);
  const sumWeights = weights.reduce((s,w)=>s+w,0);

  $("barAmount").textContent = money(bar);
  $("hostAmount").textContent = money(pool);
  $("totalHours").textContent = sumHours.toLocaleString();
  $("weightedHours").textContent = sumWeights.toLocaleString(
    "en-US",{maximumFractionDigits:3});

  let distributed = bar;
  hosts.forEach((h,i) => {
    const payout = sumWeights > 0 ? pool*weights[i]/sumWeights : 0;
    h.el.querySelector(".payout").textContent = "Payout: "+money(payout);
    h.payout = payout;
    distributed += payout;
  });

  $("results").innerHTML = hosts.map(h =>
    `<div class="stat"><span>${escapeHTML(h.name)}</span>
    <strong>${money(h.payout||0)}</strong></div>`
  ).join("") || "<p>No hosts added yet.</p>";

  $("grandTotal").textContent = money(distributed);
}

function escapeHTML(s) {
  return String(s).replace(/[&<>"']/g, c => ({
    "&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"
  }[c]));
}

function saveStaff() {
  const staff = getHosts().map(h => ({
    name: h.name==="Unnamed host"?"":h.name,
    rate: h.rate
  }));
  try {
    localStorage.setItem("tipCalcStaff", JSON.stringify(staff));
    alert("Employee names and rates saved on this device.");
  } catch(e) {
    alert("Could not save employees in this browser.");
  }
}

function loadStaff() {
  try {
    const staff = JSON.parse(localStorage.getItem("tipCalcStaff")||"[]");
    if (!staff.length) {
      alert("No saved employee list found on this device.");
      return;
    }
    $("hosts").innerHTML = "";
    staff.forEach(h=>addHost(h.name,h.rate,""));
    calculate();
  } catch(e) {
    alert("Could not load saved employees.");
  }
}

function reportHTML() {
  const total = Math.max(0,Number($("totalTips").value)||0);
  const hosts = getHosts();
  const weights = hosts.map(h=>h.hours*h.rate/100);
  const sum = weights.reduce((s,w)=>s+w,0);
  const pool = total*0.60;
  const bar = total*0.40;
  const date = $("shiftDate").value || "Not entered";
  const notes = $("shiftName").value || "No notes";
  const hostRows = hosts.map((h,i) => {
    const payout = sum>0 ? pool*weights[i]/sum : 0;
    return `<tr><td>${escapeHTML(h.name)}</td>
      <td>${h.hours}</td><td>${h.rate}%</td>
      <td>${weights[i].toFixed(3)}</td><td>${money(payout)}</td></tr>`;
  }).join("");

  return `<h1>Shift Tip Distribution Report</h1>
    <p><strong>Date:</strong> ${escapeHTML(date)}<br>
    <strong>Shift / notes:</strong> ${escapeHTML(notes)}</p>
    <h2>Tip allocation</h2>
    <table><tr><th>Category</th><th>Share</th><th>Amount</th></tr>
    <tr><td>Sushi bar</td><td>40%</td><td>${money(bar)}</td></tr>
    <tr><td>Host pool</td><td>60%</td><td>${money(pool)}</td></tr>
    <tr><td colspan="2"><strong>Total tips</strong></td><td><strong>${money(total)}</strong></td></tr>
    </table>
    <h2>Host payouts</h2>
    <table><tr><th>Employee</th><th>Hours</th><th>Rate</th>
    <th>Weighted hours</th><th>Payout</th></tr>${hostRows}</table>
    <p class="report-total">Total distributed: ${money(total)}</p>
    <p>Formula: host weighted hours = hours worked × individual tip rate ÷ 100. Each host receives a proportional share of the 60% host pool.</p>
    <p>Prepared using Shift Tip Calculator.</p>`;
}

function downloadReport() {
  $("report").innerHTML = reportHTML();
  window.print();
}

function saveRecord() {
  const record = {
    date:$("shiftDate").value,
    notes:$("shiftName").value,
    total:$("totalTips").value,
    hosts:getHosts().map(h=>({
      name:h.name,hours:h.hours,rate:h.rate,payout:h.payout||0
    })),
    savedAt:new Date().toISOString()
  };
  try {
    const records=JSON.parse(localStorage.getItem("tipCalcRecords")||"[]");
    records.unshift(record);
    localStorage.setItem("tipCalcRecords",JSON.stringify(records));
    alert("Shift record saved on this device.");
  } catch(e) {
    alert("Could not save shift record.");
  }
}

function showRecords() {
  $("recordsCard").hidden = false;
  const records=JSON.parse(localStorage.getItem("tipCalcRecords")||"[]");
  $("records").innerHTML = records.length ? records.map((r,i)=>`
    <div class="host">
      <strong>${escapeHTML(r.date||"Undated shift")}</strong>
      <p>${escapeHTML(r.notes||"")} · Tips: ${money(r.total)}</p>
      <button onclick="restoreRecord(${i})">Load shift</button>
      <button class="secondary" onclick="printRecord(${i})">Print report</button>
    </div>`).join("") : "<p>No saved shifts yet.</p>";
}

function restoreRecord(i) {
  const r=JSON.parse(localStorage.getItem("tipCalcRecords")||"[]")[i];
  if (!r) return;
  $("shiftDate").value=r.date||"";
  $("shiftName").value=r.notes||"";
  $("totalTips").value=r.total;
  $("hosts").innerHTML="";
  r.hosts.forEach(h=>addHost(h.name,h.rate,h.hours));
  calculate();
  window.scrollTo({top:0,behavior:"smooth"});
}

function printRecord(i) {
  const r=JSON.parse(localStorage.getItem("tipCalcRecords")||"[]")[i];
  if (!r) return;
  const oldDate=$("shiftDate").value;
  const oldNotes=$("shiftName").value;
  const oldTotal=$("totalTips").value;
  const oldHosts=getHosts().map(h=>({name:h.name,hours:h.hours,rate:h.rate}));
  $("shiftDate").value=r.date||"";
  $("shiftName").value=r.notes||"";
  $("totalTips").value=r.total;
  $("hosts").innerHTML="";
  r.hosts.forEach(h=>addHost(h.name,h.rate,h.hours));
  $("report").innerHTML=reportHTML();
  window.print();
  $("shiftDate").value=oldDate;
  $("shiftName").value=oldNotes;
  $("totalTips").value=oldTotal;
  $("hosts").innerHTML="";
  oldHosts.forEach(h=>addHost(h.name,h.rate,h.hours));
  calculate();
}

function clearRecords() {
  if (confirm("Permanently delete all saved shift records on this device?")) {
    localStorage.removeItem("tipCalcRecords");
    showRecords();
  }
}

$("shiftDate").value = new Date().toLocaleDateString("en-CA");
$("totalTips").addEventListener("input",calculate);
addHost("Experienced Host","100","8");
addHost("New Host","70","6");
calculate();
</script>
</body>
</html>
