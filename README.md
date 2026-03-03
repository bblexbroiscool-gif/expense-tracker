# Replace YOUR_USERNAME then run
$root = "C:\Users\Toshiba\OneDrive\Documents\expense-tracker"
Set-Location $root

# .gitignore
@'
node_modules/
build/
.env
.DS_Store
dist/
.vscode/
npm-debug.log*
yarn-error.log*
'@ | Out-File -Encoding utf8 .gitignore

# GitHub Actions workflow
$wfDir = "$root\.github\workflows"
New-Item -ItemType Directory -Force -Path $wfDir | Out-Null
@'
name: Deploy to GitHub Pages
on:
  push:
    branches: [ main ]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run build
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./build
'@ | Out-File -Encoding utf8 "$wfDir\deploy.yml"

# Update or create package.json (adds homepage, predeploy, deploy)
$pkgPath = "$root\package.json"
if (Test-Path $pkgPath) {
  $pkg = Get-Content -Raw $pkgPath | ConvertFrom-Json
} else {
  $pkg = @{ name = "expense-tracker"; version = "1.0.0"; private = $true; scripts = @{} }
}
$pkg.homepage = "https://YOUR_USERNAME.github.io/expense-tracker"
if (-not $pkg.scripts) { $pkg.scripts = @{} }
$pkg.scripts.predeploy = "npm run build"
$pkg.scripts.deploy = "gh-pages -d build"
$pkg | ConvertTo-Json -Depth 20 | Set-Content -Encoding utf8 $pkgPath

# Ensure src/App.jsx (overwrite with provided app)
$srcDir = "$root\src"
New-Item -ItemType Directory -Force -Path $srcDir | Out-Null
@'
import React, { useState } from "react";
import { motion } from "framer-motion";

export default function App() {
  const [salary, setSalary] = useState("");
  const [expenses, setExpenses] = useState([]);
  const [amount, setAmount] = useState("");
  const [note, setNote] = useState("");
  const [receipt, setReceipt] = useState(null);

  const addExpense = () => {
    if (!amount) return;
    const newExpense = {
      id: Date.now(),
      amount: parseFloat(amount),
      note,
      receiptName: receipt?.name || "No receipt",
    };
    setExpenses([newExpense, ...expenses]);
    setAmount("");
    setNote("");
    setReceipt(null);
  };

  const totalExpenses = expenses.reduce((sum, e) => sum + e.amount, 0);
  const balance = (parseFloat(salary) || 0) - totalExpenses;

  return (
    <div className="container">
      <motion.h1
        initial={{ opacity: 0, y: -10 }}
        animate={{ opacity: 1, y: 0 }}
        className="title"
      >
        Expense Tracker
      </motion.h1>

      <div className="card">
        <h2>Salary</h2>
        <input
          type="number"
          placeholder="Enter monthly salary"
          value={salary}
          onChange={(e) => setSalary(e.target.value)}
          className="input"
        />
        <div className="small">Balance: {balance.toFixed(2)}</div>
      </div>

      <div className="card">
        <h2>Add Expense</h2>
        <input
          type="number"
          placeholder="Amount"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          className="input"
        />
        <input
          placeholder="Note"
          value={note}
          onChange={(e) => setNote(e.target.value)}
          className="input"
        />
        <input
          type="file"
          onChange={(e) => setReceipt(e.target.files[0])}
          className="input"
        />
        <button onClick={addExpense} className="button">Add</button>
      </div>

      <div className="card">
        <h2>Expenses</h2>
        <div className="list">
          {expenses.map((e) => (
            <div key={e.id} className="list-item">
              <div>
                <div className="medium">{e.note || "Expense"}</div>
                <div className="tiny">{e.receiptName}</div>
              </div>
              <div>{e.amount.toFixed(2)}</div>
            </div>
          ))}
          {expenses.length === 0 && (
            <div className="small muted">No expenses yet</div>
          )}
        </div>
      </div>
    </div>
  );
}
'@ | Out-File -Encoding utf8 "$srcDir\App.jsx"

# Install gh-pages, dependencies, git init, commit, push, and deploy
npm install --save-dev gh-pages
npm install
if (-not (Test-Path ".git")) {
  git init
  git add .
  git commit -m "Prepare app for GitHub Pages deploy"
  git branch -M main
  Write-Host "Now set the remote and push (replace YOUR_USERNAME):"
  Write-Host "git remote add origin https://github.com/YOUR_USERNAME/expense-tracker.git"
  Write-Host "git push -u origin main"
} else {
  git add .
  git commit -m "Update deploy config" -q
}
# Build and deploy (user must have replaced YOUR_USERNAME and remote set)
npm run build
npm run deploy
