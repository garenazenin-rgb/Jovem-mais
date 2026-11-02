import React, { useState, useEffect, useRef, useCallback, useMemo } from "react";
import {
  Home, Wallet, CreditCard, Landmark, Send, QrCode, Key, Copy, Check, Eye, EyeOff, ShieldCheck, Lock,
  LogOut, ArrowLeft, Bell, User, BookOpen, GraduationCap, HeartHandshake, MoreHorizontal, Plus, X,
  ChevronRight, ChevronDown, AlertTriangle, Fingerprint, Smartphone, Mail, Info, Sparkles, Receipt,
  TrendingUp, TrendingDown, Clock, Trash2, RefreshCw, FlaskConical, Settings, Snowflake, Gift, Zap,
  Search, Loader2, Link2, Bookmark, Banknote, Percent, CircleCheck, CircleX,
} from "lucide-react";

// ==CORE-START==
const DB_KEY = "crtpl-jovem-db-v1";
const MAX_TRIES = 5;
const LOCK_MS = 120000;
const ALPHA = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
const DUP_MSG = "Esta identidade já está cadastrada. Escolha outro nome ou identificador.";
const MSG_OK = "Identidade CRTPL reconhecida. Cadastro autorizado!";
const MSG_FAIL = "Não foi possível reconhecer essa identidade. Confira seus dados.";
const MSG_LOCK = "Limite de tentativas atingido. Aguarde dois minutos.";

const SEEDS = [
  { name: "Marina Alves", identifier: "CRT-MARI-0001", phone: "67900000001" },
  { name: "Lucas Prado", identifier: "CRT-LUCA-0002", phone: "67900000002" },
  { name: "Beatriz Santos", identifier: "CRT-BEAT-0003", phone: "67900000003" },
  { name: "Rafael Monteiro", identifier: "CRT-RAFA-0004", phone: "67900000004" },
  { name: "Ana Clara Ribeiro", identifier: "CRT-ANAC-0005", phone: "67900000005" },
];

const emptyDB = () => ({ v: 1, identities: [], accounts: [], locks: {} });
const stripAcc = (s) => (s || "").normalize("NFD").replace(/[\u0300-\u036f]/g, "");
const cleanName = (s) => (s || "").trim().replace(/\s+/g, " ");
const normName = (s) => stripAcc(cleanName(s)).toLowerCase();
const slug = (s) => stripAcc(s).toLowerCase().replace(/[^a-z0-9]/g, "");
const digitsOf = (s) => (s || "").replace(/\D/g, "");
const normPhone = (s) => {
  let d = digitsOf(s);
  if (d.length > 11 && d.startsWith("55")) d = d.slice(2);
  return d;
};
const rnd = (n) => {
  if (typeof crypto !== "undefined" && crypto.getRandomValues) {
    const a = new Uint32Array(1);
    crypto.getRandomValues(a);
    return a[0] % n;
  }
  return Math.floor(Math.random() * n);
};
const rndStr = (n) => Array.from({ length: n }, () => ALPHA[rnd(ALPHA.length)]).join("");
const rndDigits = (n) => Array.from({ length: n }, () => rnd(10)).join("");
const cap = (s) => (s ? s.charAt(0).toUpperCase() + s.slice(1).toLowerCase() : "");
const firstName = (n) => cap(cleanName(n).split(" ")[0] || "");
const uid = (p) => p + Date.now().toString(36) + rndStr(4).toLowerCase();
const seedEmail = (s) => slug(s.name) + "@jvpl.demo";

const maskIdent = (id) => (id ? "CRT-••••-" + id.slice(-4) : "—");
const fmtPhone = (d) => (d && d.length === 11 ? `+55 (${d.slice(0, 2)}) ${d.slice(2, 7)}-${d.slice(7)}` : d || "");
const brl = (c) => (c / 100).toLocaleString("pt-BR", { style: "currency", currency: "BRL" });
const fmtDate = (t) =>
  new Date(t).toLocaleString("pt-BR", { day: "2-digit", month: "2-digit", year: "numeric", hour: "2-digit", minute: "2-digit" });
const fmtDay = (t) => new Date(t).toLocaleDateString("pt-BR", { day: "2-digit", month: "short" });
const mmss = (ms) => {
  const s = Math.max(0, Math.ceil(ms / 1000));
  return String(Math.floor(s / 60)).padStart(2, "0") + ":" + String(s % 60).padStart(2, "0");
};

async function hashPass(pass, salt) {
  try {
    const enc = new TextEncoder();
    const k = await crypto.subtle.importKey("raw", enc.encode(pass), "PBKDF2", false, ["deriveBits"]);
    const bits = await crypto.subtle.deriveBits(
      { name: "PBKDF2", salt: enc.encode(salt), iterations: 50000, hash: "SHA-256" },
      k,
      256
    );
    return "pbkdf2:" + Array.from(new Uint8Array(bits)).map((b) => b.toString(16).padStart(2, "0")).join("");
  } catch (e) {
    // Fallback demonstrativo (sem WebCrypto): hash simples, NÃO seguro.
    let h1 = 2166136261, h2 = 5381;
    const s = salt + "|" + pass;
    for (let r = 0; r < 200; r++)
      for (let i = 0; i < s.length; i++) {
        h1 = Math.imul(h1 ^ s.charCodeAt(i), 16777619);
        h2 = (Math.imul(h2, 33) + s.charCodeAt(i)) | 0;
      }
    return "weak:" + (h1 >>> 0).toString(16) + (h2 >>> 0).toString(16);
  }
}
const newSalt = () => rndStr(16);

/* ---------- Bloqueio por tentativas (controlado pelo sistema) ---------- */
function lockInfo(db, bucket, now) {
  const l = db.locks[bucket] || { fails: 0, until: 0 };
  if (l.until && l.until > now) return { locked: true, remaining: l.until - now, left: 0 };
  const fails = l.until && l.until <= now ? 0 : l.fails;
  return { locked: false, remaining: 0, left: MAX_TRIES - fails };
}
function addFail(db, bucket, now) {
  let l = db.locks[bucket] || { fails: 0, until: 0 };
  if (l.until && l.until <= now) l = { fails: 0, until: 0 };
  l = { ...l, fails: l.fails + 1 };
  if (l.fails >= MAX_TRIES) l.until = now + LOCK_MS;
  return { ...db, locks: { ...db.locks, [bucket]: l } };
}
function clearLock(db, bucket) {
  const locks = { ...db.locks };
  delete locks[bucket];
  return { ...db, locks };
}
function failResult(db, bucket, now) {
  const ndb = addFail(db, bucket, now);
  const li = lockInfo(ndb, bucket, now);
  return { db: ndb, status: li.locked ? "locked" : "fail", remaining: li.remaining, left: li.left };
}

/* ---------- Identidades CRTPL ---------- */
function createIdentity(db, { name, passHash, salt }, now) {
  const nm = cleanName(name);
  const nn = normName(nm);
  const sg = slug(nm);
  if (!sg) return { error: "Use letras no nome." };
  const email = sg + "@jvpl.demo";
  if (
    db.identities.some((i) => i.normName === nn || i.email === email) ||
    SEEDS.some((s) => normName(s.name) === nn || seedEmail(s) === email)
  )
    return { error: DUP_MSG };
  let identifier, tries = 0;
  do {
    identifier = "CRT-" + rndStr(4) + "-" + rndStr(4);
    if (++tries > 500) return { error: DUP_MSG };
  } while (db.identities.some((i) => i.identifier === identifier));
  let phone;
  tries = 0;
  do {
    phone = "6790" + rndDigits(7);
    if (++tries > 500) return { error: DUP_MSG };
  } while (db.identities.some((i) => i.phone === phone) || SEEDS.some((s) => s.phone === phone));
  const used = new Set(db.identities.map((i) => i.cdg));
  if (used.size >= 10000) return { error: "Todos os CDGs de 4 dígitos já foram usados nesta base." };
  let cdg;
  do cdg = String(rnd(10000)).padStart(4, "0");
  while (used.has(cdg));
  const identity = {
    id: uid("id_"),
    name: nm,
    normName: nn,
    identifier,
    email,
    phone,
    ddd: "67",
    carrier: "Claro",
    cdg,
    passHash,
    salt,
    createdAt: now,
    status: "Ativa",
  };
  return { db: { ...db, identities: [...db.identities, identity] }, identity };
}

function findIdentity(db, key) {
  const k = cleanName(key);
  if (!k) return null;
  const nn = normName(k), idU = k.toUpperCase(), em = k.toLowerCase();
  return db.identities.find((i) => i.normName === nn || i.identifier === idU || i.email === em) || null;
}
const identOf = (db, acct) => db.identities.find((i) => i.id === acct.identityId);
const accountOf = (db, identityId) => db.accounts.find((a) => a.identityId === identityId) || null;

function dataMatches(ident, data) {
  return !!(
    ident &&
    ident.normName === normName(data.name) &&
    ident.identifier === (data.identifier || "").trim().toUpperCase() &&
    ident.email === (data.email || "").trim().toLowerCase() &&
    ident.phone === normPhone(data.phone)
  );
}
function lookupByIdentifier(db, data) {
  return db.identities.find((i) => i.identifier === (data.identifier || "").trim().toUpperCase());
}

/* Etapa 1 do JOVEM+: nome, identificador, e-mail e telefone precisam ser da MESMA identidade. */
function verifyData(db, data, now, bucket = "jovem-verify") {
  const li = lockInfo(db, bucket, now);
  if (li.locked) return { db, status: "locked", remaining: li.remaining, left: 0 };
  const ident = lookupByIdentifier(db, data);
  if (dataMatches(ident, data)) return { db, status: "ok", identity: ident, left: li.left };
  return failResult(db, bucket, now);
}
/* Etapa 2: revalida TUDO + CDG contra o registro original. */
function verifyIdentity(db, data, cdg, now, bucket = "jovem-verify") {
  const li = lockInfo(db, bucket, now);
  if (li.locked) return { db, status: "locked", remaining: li.remaining, left: 0 };
  const ident = lookupByIdentifier(db, data);
  if (dataMatches(ident, data) && /^\d{4}$/.test(cdg || "") && ident.cdg === cdg)
    return { db: clearLock(db, bucket), status: "ok", identity: ident };
  return failResult(db, bucket, now);
}
/* Confirmação por CDG para recuperação de senha. */
function verifyCDG(db, ident, cdg, now, bucket) {
  const li = lockInfo(db, bucket, now);
  if (li.locked) return { db, status: "locked", remaining: li.remaining, left: 0 };
  if (ident && /^\d{4}$/.test(cdg || "") && ident.cdg === cdg) return { db: clearLock(db, bucket), status: "ok", identity: ident };
  return failResult(db, bucket, now);
}

/* ---------- Contas JOVEM+ ---------- */
function createAccount(db, identityId, passHash, salt, now) {
  const ident = db.identities.find((i) => i.id === identityId);
  if (!ident) return { error: "Identidade CRTPL não encontrada." };
  if (accountOf(db, identityId)) return { error: "Esta identidade CRTPL já possui uma conta JOVEM+. Use Entrar." };
  let vid, tries = 0;
  do {
    vid = "VRT-" + rndStr(4) + "-" + rndStr(4);
    if (++tries > 500) return { error: "Falha ao gerar cartão virtual." };
  } while (db.accounts.some((a) => a.cards.virtual.id === vid));
  let number;
  do number = `${10000 + rnd(90000)}-${rnd(10)}`;
  while (db.accounts.some((a) => a.number === number));
  const yr = new Date(now).getFullYear() + 4;
  const acct = {
    id: uid("jv_"),
    identityId,
    passHash,
    salt,
    createdAt: now,
    agency: "0001",
    number,
    balance: 150000,
    creditLimit: 300000,
    purchases: [],
    txs: [
      { id: uid("t"), type: "in", kind: "init", title: "Crédito inicial demonstrativo", counter: { name: "JOVEM+ (simulação)", idMasked: "—" }, amount: 150000, desc: "Saldo fictício de boas-vindas", date: now, receipt: "SIM" + rndStr(12) },
    ],
    rndKeys: [],
    cards: {
      debit: { blocked: false },
      credit: { blocked: false },
      virtual: { id: vid, exp: `12/${String(yr).slice(2)}`, blocked: false },
    },
    notifs: [
      { id: uid("n"), title: "Conta criada", text: "Sua conta demonstrativa JOVEM+ foi criada. Nada aqui movimenta dinheiro real.", date: now, read: false },
      { id: uid("n"), title: "Dica de segurança", text: "Nunca informe senhas ou códigos reais em apps de demonstração.", date: now, read: false },
    ],
    donations: [],
    saved: [],
    notifOn: true,
  };
  return { db: { ...db, accounts: [...db.accounts, acct] }, account: acct };
}
const patchAcct = (db, id, fn) => ({ ...db, accounts: db.accounts.map((a) => (a.id === id ? fn(a) : a)) });

/* ---------- Pix demonstrativo ---------- */
function directory(db) {
  const out = SEEDS.map((s) => ({
    kind: "seed",
    name: s.name,
    identifier: s.identifier,
    keys: [seedEmail(s), s.phone, s.identifier],
  }));
  db.accounts.forEach((a) => {
    const i = identOf(db, a);
    if (i) out.push({ kind: "account", accountId: a.id, name: i.name, identifier: i.identifier, keys: [i.email, i.phone, i.identifier, ...a.rndKeys] });
  });
  return out;
}
function findRecipient(db, selfId, input) {
  const k = cleanName(input);
  if (!k) return null;
  const cands = [k.toLowerCase(), normPhone(k), k.toUpperCase()];
  return directory(db).find((d) => d.accountId !== selfId && d.keys.some((x) => cands.includes(x))) || null;
}
function sendPix(db, accId, rec, amount, desc, now) {
  const a = db.accounts.find((x) => x.id === accId);
  if (!a) return { error: "Conta não encontrada." };
  if (!(amount > 0)) return { error: "Informe um valor maior que zero." };
  if (rec.accountId === accId) return { error: "Não é possível enviar Pix para você mesmo." };
  if (amount > a.balance) return { error: "Saldo insuficiente para esta simulação." };
  const me = identOf(db, a);
  const receipt = "SIM" + rndStr(12);
  const tx = { id: uid("t"), type: "out", kind: "pix", title: "Pix enviado", counter: { name: rec.name, idMasked: maskIdent(rec.identifier) }, amount, desc: desc || "", date: now, receipt };
  const accounts = db.accounts.map((x) => {
    if (x.id === accId) return { ...x, balance: x.balance - amount, txs: [tx, ...x.txs] };
    if (rec.accountId && x.id === rec.accountId) {
      const inTx = { ...tx, id: uid("t"), type: "in", title: "Pix recebido", counter: { name: me.name, idMasked: maskIdent(me.identifier) } };
      const n = { id: uid("n"), title: "Pix recebido (simulação)", text: `${me.name} enviou ${brl(amount)}.`, date: now, read: false };
      return { ...x, balance: x.balance + amount, txs: [inTx, ...x.txs], notifs: [n, ...x.notifs] };
    }
    return x;
  });
  return { db: { ...db, accounts }, tx, balanceAfter: a.balance - amount };
}
function receivePixSim(db, accId, amount, from, now) {
  if (!(amount > 0)) return { error: "Informe um valor para simular o recebimento." };
  const tx = { id: uid("t"), type: "in", kind: "pix", title: "Pix recebido", counter: { name: from.name, idMasked: maskIdent(from.identifier) }, amount, desc: "Cobrança demonstrativa", date: now, receipt: "SIM" + rndStr(12) };
  const n = { id: uid("n"), title: "Pix recebido (simulação)", text: `${from.name} pagou ${brl(amount)}.`, date: now, read: false };
  return { db: patchAcct(db, accId, (a) => ({ ...a, balance: a.balance + amount, txs: [tx, ...a.txs], notifs: [n, ...a.notifs] })), tx };
}
function addRandomKey(db, accId) {
  const a = db.accounts.find((x) => x.id === accId);
  if (!a) return { error: "Conta não encontrada." };
  if (a.rndKeys.length >= 3) return { error: "Limite de 3 chaves aleatórias demonstrativas." };
  let key;
  do key = "chave-" + rndStr(8).toLowerCase() + "-" + rndStr(4).toLowerCase();
  while (db.accounts.some((x) => x.rndKeys.includes(key)));
  return { db: patchAcct(db, accId, (x) => ({ ...x, rndKeys: [...x.rndKeys, key] })), key };
}
const removeRandomKey = (db, accId, key) => patchAcct(db, accId, (a) => ({ ...a, rndKeys: a.rndKeys.filter((k) => k !== key) }));

/* ---------- Crédito ---------- */
const FREE_INSTALLMENTS = 3;
const MONTHLY_RATE = 0.0199;
function calcInstallment(P, n) {
  if (n <= FREE_INSTALLMENTS) {
    const m = Math.round(P / n);
    return { monthly: m, total: m * n, rate: 0 };
  }
  const pmt = (P * MONTHLY_RATE) / (1 - Math.pow(1 + MONTHLY_RATE, -n));
  const m = Math.round(pmt);
  return { monthly: m, total: m * n, rate: MONTHLY_RATE };
}
const usedCredit = (a) => a.purchases.reduce((s, p) => s + (p.n - p.paid) * p.monthly, 0);
function addPurchase(db, accId, desc, P, n, now) {
  const a = db.accounts.find((x) => x.id === accId);
  if (!a) return { error: "Conta não encontrada." };
  if (a.cards.credit.blocked) return { error: "Cartão de crédito bloqueado. Desbloqueie em Cartões." };
  if (!(P > 0)) return { error: "Informe o valor da compra." };
  const c = calcInstallment(P, n);
  if (c.total > a.creditLimit - usedCredit(a)) return { error: "Limite disponível insuficiente para esta compra." };
  const p = { id: uid("p"), desc, base: P, n, paid: 0, monthly: c.monthly, total: c.total, date: now };
  return { db: patchAcct(db, accId, (x) => ({ ...x, purchases: [p, ...x.purchases] })) };
}
function payInvoice(db, accId, now) {
  const a = db.accounts.find((x) => x.id === accId);
  const open = a.purchases.filter((p) => p.paid < p.n);
  const due = open.reduce((s, p) => s + p.monthly, 0);
  if (!due) return { error: "Não há parcelas em aberto." };
  if (due > a.balance) return { error: "Saldo insuficiente para pagar a fatura." };
  const tx = { id: uid("t"), type: "out", kind: "fatura", title: "Fatura do cartão (simulação)", counter: { name: "JOVEM+ Crédito", idMasked: "—" }, amount: due, desc: "Parcela do mês", date: now, receipt: "SIM" + rndStr(12) };
  return { db: patchAcct(db, accId, (x) => ({ ...x, balance: x.balance - due, txs: [tx, ...x.txs], purchases: x.purchases.map((p) => (p.paid < p.n ? { ...p, paid: p.paid + 1 } : p)) })), tx, due };
}
function anticipateQuote(p) {
  const rem = p.n - p.paid;
  const value = rem * p.monthly;
  const pct = Math.min(0.005 * rem, 0.1);
  const discount = Math.round(value * pct);
  return { rem, value, pct, discount, pay: value - discount };
}
function anticipate(db, accId, pid, now) {
  const a = db.accounts.find((x) => x.id === accId);
  const p = a.purchases.find((x) => x.id === pid);
  if (!p || p.paid >= p.n) return { error: "Compra já quitada." };
  const q = anticipateQuote(p);
  if (q.pay > a.balance) return { error: "Saldo insuficiente para antecipar." };
  const tx = { id: uid("t"), type: "out", kind: "fatura", title: "Antecipação de parcelas (simulação)", counter: { name: p.desc, idMasked: "—" }, amount: q.pay, desc: `${q.rem} parcela(s) com desconto de ${brl(q.discount)}`, date: now, receipt: "SIM" + rndStr(12) };
  return { db: patchAcct(db, accId, (x) => ({ ...x, balance: x.balance - q.pay, txs: [tx, ...x.txs], purchases: x.purchases.map((y) => (y.id === pid ? { ...y, paid: y.n } : y)) })), tx, q };
}

/* ---------- Cartões, doações ---------- */
function setCardBlocked(db, accId, card, blocked) {
  return patchAcct(db, accId, (a) => ({ ...a, cards: { ...a.cards, [card]: { ...a.cards[card], blocked } } }));
}
function debitSpend(db, accId, desc, amount, now) {
  const a = db.accounts.find((x) => x.id === accId);
  if (a.cards.debit.blocked) return { error: "Cartão de débito bloqueado." };
  if (!(amount > 0)) return { error: "Informe o valor." };
  if (amount > a.balance) return { error: "Saldo insuficiente." };
  const tx = { id: uid("t"), type: "out", kind: "card", title: "Compra no débito (simulação)", counter: { name: desc, idMasked: "—" }, amount, desc: "Cartão demonstrativo", date: now, receipt: "SIM" + rndStr(12) };
  return { db: patchAcct(db, accId, (x) => ({ ...x, balance: x.balance - amount, txs: [tx, ...x.txs] })), tx };
}
function donate(db, accId, cause, amount, now) {
  const a = db.accounts.find((x) => x.id === accId);
  if (!(amount > 0)) return { error: "Informe o valor da doação." };
  if (amount > a.balance) return { error: "Saldo insuficiente." };
  const tx = { id: uid("t"), type: "out", kind: "donation", title: "Doação simulada", counter: { name: cause.name, idMasked: "—" }, amount, desc: "Nenhum valor real foi doado", date: now, receipt: "SIM" + rndStr(12) };
  const d = { id: uid("d"), causeId: cause.id, cause: cause.name, amount, date: now };
  return { db: patchAcct(db, accId, (x) => ({ ...x, balance: x.balance - amount, txs: [tx, ...x.txs], donations: [d, ...x.donations] })), tx };
}

/* ---------- Testes automáticos da lógica ---------- */
function runTests() {
  const res = [];
  const t = (name, fn) => {
    try {
      const r = fn();
      res.push({ name, ok: r === true || r === undefined, detail: typeof r === "string" ? r : "" });
    } catch (e) {
      res.push({ name, ok: false, detail: String(e && e.message ? e.message : e) });
    }
  };
  const NOW = 1_700_000_000_000;
  let db = emptyDB();
  let A, B;
  const mk = (nm) => {
    const r = createIdentity(db, { name: nm, passHash: "h", salt: "s" }, NOW);
    if (r.error) return r;
    db = r.db;
    return r;
  };
  t("1. Cria identidade CRTPL e gera identificadores", () => {
    const r = mk("Cleitindo Grau");
    A = r.identity;
    if (!/^CRT-[A-Z2-9]{4}-[A-Z2-9]{4}$/.test(A.identifier)) return "identificador inválido";
    if (A.email !== "cleitindograu@jvpl.demo") return "e-mail inválido: " + A.email;
    if (!/^6790\d{7}$/.test(A.phone) || A.ddd !== "67" || A.carrier !== "Claro") return "telefone inválido";
    if (!/^\d{4}$/.test(A.cdg)) return "CDG inválido";
  });
  t("2. Bloqueia nome duplicado (maiúsculas/espaços/acentos)", () => {
    const r1 = createIdentity(db, { name: "  CLEITINDO    grau ", passHash: "h", salt: "s" }, NOW);
    const r2 = createIdentity(db, { name: "cleitíndo grau", passHash: "h", salt: "s" }, NOW);
    if (r1.error !== DUP_MSG || r2.error !== DUP_MSG) return "duplicação não impedida";
    if (db.identities.length !== 1) return "base alterada";
  });
  t("3. Segunda identidade tem identificadores exclusivos", () => {
    B = mk("Maria Souza").identity;
    if (A.cdg === B.cdg || A.identifier === B.identifier || A.email === B.email || A.phone === B.phone) return "colisão";
  });
  t("4. 300 identidades: nenhum CDG/identificador/telefone repetido", () => {
    let d2 = emptyDB();
    const n = (i) => "Teste " + String(i).padStart(3, "0").replace(/[0-9]/g, (c) => "abcdefghij"[c]);
    for (let i = 0; i < 300; i++) {
      const r = createIdentity(d2, { name: n(i), passHash: "h", salt: "s" }, NOW);
      if (r.error) return "erro na criação " + i + ": " + r.error;
      d2 = r.db;
    }
    const u = (f) => new Set(d2.identities.map((x) => x[f])).size;
    if (u("cdg") !== 300 || u("identifier") !== 300 || u("phone") !== 300 || u("email") !== 300) return "duplicado encontrado";
  });
  const dataA = () => ({ name: A.name, identifier: A.identifier, email: A.email, phone: fmtPhone(A.phone) });
  t("5. CDG correto libera o cadastro", () => {
    const r = verifyIdentity(db, dataA(), A.cdg, NOW);
    if (r.status !== "ok" || r.identity.id !== A.id) return "status " + r.status;
  });
  t("6. CDG incorreto nega e mostra tentativas restantes", () => {
    const wrong = A.cdg === "0000" ? "0001" : "0000";
    const r = verifyIdentity(db, dataA(), wrong, NOW);
    if (r.status !== "fail" || r.left !== 4) return `status ${r.status}, left ${r.left}`;
  });
  t("7. Misturar dados de identidades diferentes é negado", () => {
    const mixes = [
      [{ ...dataA(), identifier: B.identifier }, A.cdg],
      [{ ...dataA(), email: B.email }, A.cdg],
      [{ ...dataA(), phone: B.phone }, A.cdg],
      [{ ...dataA(), name: B.name }, A.cdg],
      [dataA(), B.cdg],
      [{ name: B.name, identifier: B.identifier, email: B.email, phone: B.phone }, A.cdg],
    ];
    for (const [d, c] of mixes) if (verifyIdentity(db, d, c, NOW).status === "ok") return "mistura aceita!";
  });
  t("8. Cinco erros bloqueiam por 2 min; depois libera", () => {
    let d = db;
    const wrong = A.cdg === "0000" ? "0001" : "0000";
    let last;
    for (let i = 0; i < 5; i++) {
      last = verifyIdentity(d, dataA(), wrong, NOW + i);
      d = last.db;
    }
    if (last.status !== "locked") return "não bloqueou";
    const during = verifyIdentity(d, dataA(), A.cdg, NOW + 60000);
    if (during.status !== "locked") return "CDG correto passou durante o bloqueio";
    const after = verifyIdentity(d, dataA(), A.cdg, NOW + 4 + LOCK_MS + 1);
    if (after.status !== "ok") return "não liberou após 2 min";
    const afterWrong = verifyIdentity(d, dataA(), wrong, NOW + 4 + LOCK_MS + 1);
    if (afterWrong.status !== "fail" || afterWrong.left !== 4) return "tentativas não reiniciaram";
  });
  let acc;
  t("9. Uma identidade não cria duas contas JOVEM+", () => {
    const r = createAccount(db, A.id, "h2", "s2", NOW);
    if (r.error) return r.error;
    db = r.db;
    acc = r.account;
    const r2 = createAccount(db, A.id, "h3", "s3", NOW);
    if (!r2.error) return "segunda conta criada";
  });
  t("10. Pix: valida saldo, destinatário e atualiza saldos", () => {
    const rec = findRecipient(db, acc.id, "marinaalves@jvpl.demo");
    if (!rec) return "destinatário não encontrado";
    if (findRecipient(db, acc.id, A.email)) return "permitiu achar a si mesmo";
    if (!sendPix(db, acc.id, rec, 99999999, "x", NOW).error) return "aceitou sem saldo";
    const r = sendPix(db, acc.id, rec, 25050, "teste", NOW);
    if (r.error || r.balanceAfter !== 150000 - 25050) return "saldo errado";
    db = r.db;
  });
  t("11. Pix entre duas contas JOVEM+ credita o destinatário", () => {
    const rb = createAccount(db, B.id, "h", "s", NOW);
    db = rb.db;
    const rec = findRecipient(db, acc.id, B.identifier);
    if (!rec || !rec.accountId) return "conta B não aparece no diretório";
    const r = sendPix(db, acc.id, rec, 1000, "", NOW);
    db = r.db;
    const b = db.accounts.find((x) => x.id === rb.account.id);
    if (b.balance !== 151000) return "destinatário não creditado";
  });
  t("12. Crédito: respeita limite, antecipação e fatura", () => {
    let r = addPurchase(db, acc.id, "Compra", 100000, 5, NOW);
    if (r.error) return r.error;
    db = r.db;
    if (!addPurchase(db, acc.id, "Grande", 900000, 2, NOW).error) return "passou do limite";
    const a1 = db.accounts.find((x) => x.id === acc.id);
    if (usedCredit(a1) <= 0) return "uso não calculado";
    const pi = payInvoice(db, acc.id, NOW);
    if (pi.error) return pi.error;
    db = pi.db;
    const a2 = db.accounts.find((x) => x.id === acc.id);
    const an = anticipate(db, acc.id, a2.purchases[0].id, NOW);
    if (an.error) return an.error;
    db = an.db;
    if (usedCredit(db.accounts.find((x) => x.id === acc.id)) !== 0) return "antecipação não quitou";
  });
  t("13. Recuperação por CDG também tem limite de tentativas", () => {
    let d = db;
    let r;
    for (let i = 0; i < 5; i++) {
      r = verifyCDG(d, A, "9999" === A.cdg ? "9998" : "9999", NOW + i, "recover-test");
      d = r.db;
    }
    if (r.status !== "locked") return "não bloqueou";
  });
  return res;
}
// ==CORE-END==
const CSS = `
*{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
.shell{min-height:100vh;display:flex;justify-content:center;background:#0b0912;font-family:"Avenir Next","Segoe UI",Roboto,system-ui,sans-serif}
.phone{width:100%;max-width:430px;height:100vh;height:100dvh;position:relative;overflow:hidden;display:flex;flex-direction:column;background:var(--bg);color:var(--tx)}
@media(min-width:520px){.shell{padding:18px 0;align-items:center}.phone{height:min(880px,calc(100vh - 36px));border-radius:36px;box-shadow:0 30px 90px #000a,0 0 0 1px #ffffff14}}
button{font:inherit;cursor:pointer;border:0;color:inherit;background:none;padding:0}
button:disabled{cursor:not-allowed;opacity:.55}
input,select,textarea{font:inherit;color:inherit}
:focus-visible{outline:2px solid var(--ac);outline-offset:2px}
.scroll{flex:1;overflow-y:auto;overflow-x:hidden;padding:18px 18px 28px;scrollbar-width:none}
.scroll::-webkit-scrollbar{display:none}
.row{display:flex;align-items:center}.col{display:flex;flex-direction:column}
.g4{gap:4px}.g8{gap:8px}.g12{gap:12px}.g16{gap:16px}.grow{flex:1;min-width:0}.sb{justify-content:space-between}.ctr{justify-content:center;text-align:center}
.mt8{margin-top:8px}.mt12{margin-top:12px}.mt16{margin-top:16px}.mt24{margin-top:24px}
.mut{color:var(--mut)}.sm{font-size:13px}.xs{font-size:11.5px}.b{font-weight:700}.ell{overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
h1,h2,h3,p{margin:0}
h1{font-size:28px;line-height:1.12;letter-spacing:-.02em}h2{font-size:20px;letter-spacing:-.01em}h3{font-size:15.5px}

/* tema CRTPL */
.cr{--bg:#120E20;--s1:#1C1630;--s2:#2A2147;--tx:#F4F0FF;--mut:#A99FC9;--ac:#FFC24B;--onac:#2A1B00;--ac2:#8F7BFF;--ok:#5FE3A1;--er:#FF7A88;--ln:#ffffff1c;--sh:0 10px 30px #0006}
/* tema JOVEM+ */
.jv{--bg:#F2F5FA;--s1:#FFFFFF;--s2:#E6EDF7;--tx:#0B2447;--mut:#5C6F8A;--ac:#1F6FEB;--onac:#FFFFFF;--ac2:#0B2447;--ok:#0E9F5D;--er:#D9363E;--ln:#0b244716;--sh:0 8px 24px #0b24471a}

.btn{display:flex;align-items:center;justify-content:center;gap:8px;width:100%;padding:14px 18px;border-radius:14px;font-weight:700;font-size:15.5px;background:var(--ac);color:var(--onac);transition:transform .12s,filter .15s}
.btn:active:not(:disabled){transform:scale(.97)}.btn:hover:not(:disabled){filter:brightness(1.07)}
.btn.sec{background:var(--s2);color:var(--tx)}
.btn.out{background:transparent;color:var(--tx);box-shadow:inset 0 0 0 1.5px var(--ln)}
.btn.danger{background:transparent;color:var(--er);box-shadow:inset 0 0 0 1.5px var(--er)}
.btn.sm2{padding:10px 14px;font-size:13.5px;border-radius:11px;width:auto}
.link{color:var(--ac);font-weight:600;font-size:14px;text-decoration:underline;text-underline-offset:3px}
.cr .link{color:var(--ac)}.jv .link{color:var(--ac)}
.card{background:var(--s1);border-radius:20px;padding:16px;box-shadow:var(--sh)}
.cr .card{box-shadow:inset 0 0 0 1px var(--ln)}
.field{display:flex;flex-direction:column;gap:6px;margin-top:14px}
.field>label{font-size:13px;font-weight:600;color:var(--mut)}
.inw{position:relative;display:flex;align-items:center}
.inp{width:100%;padding:14px 14px;border-radius:13px;background:var(--s1);border:1.5px solid var(--ln);outline:none;font-size:16px;transition:border-color .15s,box-shadow .15s}
.cr .inp{background:var(--s1)}
.inp:focus{border-color:var(--ac);box-shadow:0 0 0 3px color-mix(in srgb,var(--ac) 25%,transparent)}
.inp.bad{border-color:var(--er)}
.inp.pad{padding-right:46px}
.eye{position:absolute;right:6px;width:38px;height:38px;display:grid;place-items:center;border-radius:10px;color:var(--mut)}
.err{color:var(--er);font-size:13px;display:flex;gap:6px;align-items:flex-start;line-height:1.35}
.okmsg{color:var(--ok);font-size:13px;display:flex;gap:6px;align-items:flex-start}
.banner{display:flex;gap:10px;padding:11px 13px;border-radius:14px;font-size:12.5px;line-height:1.4;background:color-mix(in srgb,var(--ac) 14%,transparent);color:var(--tx);box-shadow:inset 0 0 0 1px color-mix(in srgb,var(--ac) 35%,transparent)}
.banner svg{flex:none;margin-top:1px;color:var(--ac)}
.banner.er{background:color-mix(in srgb,var(--er) 13%,transparent);box-shadow:inset 0 0 0 1px color-mix(in srgb,var(--er) 45%,transparent)}.banner.er svg{color:var(--er)}
.banner.ok{background:color-mix(in srgb,var(--ok) 14%,transparent);box-shadow:inset 0 0 0 1px color-mix(in srgb,var(--ok) 45%,transparent)}.banner.ok svg{color:var(--ok)}
.back{display:flex;align-items:center;gap:6px;color:var(--mut);font-weight:600;font-size:14px;padding:6px 0;margin-bottom:8px}
.chip{padding:8px 13px;border-radius:99px;background:var(--s2);font-size:13px;font-weight:600;white-space:nowrap;transition:background .15s,color .15s}
.chip.on{background:var(--ac);color:var(--onac)}
.chips{display:flex;gap:8px;overflow-x:auto;padding:2px 0 6px;scrollbar-width:none}.chips::-webkit-scrollbar{display:none}
.badge{display:inline-flex;align-items:center;gap:5px;padding:3px 9px;border-radius:99px;font-size:11.5px;font-weight:700;background:color-mix(in srgb,var(--ok) 18%,transparent);color:var(--ok)}
.badge.warn{background:color-mix(in srgb,var(--ac) 20%,transparent);color:var(--ac)}.badge.er{background:color-mix(in srgb,var(--er) 18%,transparent);color:var(--er)}
.irow{display:flex;align-items:center;gap:12px;padding:13px 0;border-bottom:1px solid var(--ln)}.irow:last-child{border-bottom:0}
.ico{width:40px;height:40px;border-radius:12px;display:grid;place-items:center;background:var(--s2);flex:none}
.sep{height:1px;background:var(--ln);margin:14px 0}

/* sheet / modal */
.ovl{position:absolute;inset:0;background:#000a;display:flex;align-items:flex-end;z-index:50;animation:fade .2s}
.sheet{width:100%;max-height:92%;overflow-y:auto;background:var(--bg);border-radius:26px 26px 0 0;padding:10px 18px 24px;animation:up .28s cubic-bezier(.2,.9,.3,1);scrollbar-width:none}
.sheet::-webkit-scrollbar{display:none}
.grab{width:42px;height:4px;border-radius:9px;background:var(--ln);margin:0 auto 14px}
.dlg{margin:auto 18px;background:var(--s1);border-radius:22px;padding:20px;animation:pop .22s cubic-bezier(.2,1.2,.4,1);box-shadow:var(--sh)}
.toast{position:absolute;top:14px;left:14px;right:14px;z-index:90;padding:12px 14px;border-radius:14px;display:flex;gap:10px;align-items:center;font-size:14px;font-weight:600;background:var(--s1);color:var(--tx);box-shadow:0 12px 30px #0007,inset 0 0 0 1px var(--ln);animation:drop .3s cubic-bezier(.2,1,.3,1)}
.toast.ok svg{color:var(--ok)}.toast.er svg{color:var(--er)}.toast.in svg{color:var(--ac)}

.spin{animation:spin 1s linear infinite}
.shake{animation:shake .4s}
@keyframes spin{to{transform:rotate(360deg)}}
@keyframes fade{from{opacity:0}}
@keyframes up{from{transform:translateY(40px);opacity:0}}
@keyframes drop{from{transform:translateY(-20px);opacity:0}}
@keyframes pop{from{transform:scale(.9);opacity:0}}
@keyframes shake{20%,60%{transform:translateX(-6px)}40%,80%{transform:translateX(6px)}}
@keyframes rise{from{transform:translateY(14px);opacity:0}}
@keyframes flip{from{transform:rotateX(70deg) translateY(30px);opacity:0}}
@keyframes sheen{from{background-position:-200% 0}to{background-position:200% 0}}
@keyframes bob{50%{transform:translateY(-6px)}}
@keyframes ring{0%{transform:scale(.6);opacity:.9}100%{transform:scale(1.7);opacity:0}}
.rise{animation:rise .5s cubic-bezier(.2,.9,.3,1) both}
@media(prefers-reduced-motion:reduce){*{animation-duration:.01ms!important;animation-iteration-count:1!important;transition:none!important}}

/* ===== CRTPL ===== */
.cr .hero{position:relative;padding:30px 22px 18px;overflow:hidden}
.cr .glow{position:absolute;width:300px;height:300px;border-radius:50%;filter:blur(70px);opacity:.38;pointer-events:none}
.idcard{position:relative;border-radius:24px;padding:20px;overflow:hidden;color:#150F26;background:linear-gradient(135deg,#FFD97A 0%,#FFC24B 38%,#F29B3C 100%);box-shadow:0 22px 40px #0008,inset 0 0 0 1px #fff6;animation:flip .7s cubic-bezier(.2,.9,.3,1) both;transform-origin:center bottom}
.idcard::after{content:"";position:absolute;inset:0;background:linear-gradient(115deg,transparent 30%,#ffffff66 48%,transparent 62%);background-size:200% 100%;animation:sheen 4.5s linear infinite;pointer-events:none}
.idcard .guil{position:absolute;right:-60px;top:-60px;opacity:.22}
.idcard .lab{font-size:10.5px;font-weight:700;opacity:.65;letter-spacing:.05em}
.idcard .val{font-weight:800;font-size:15px}
.stamp{position:absolute;right:14px;bottom:12px;font-size:10px;font-weight:800;letter-spacing:.14em;border:1.5px solid #150F26aa;border-radius:6px;padding:2px 7px;opacity:.7;transform:rotate(-6deg)}
.cdg{display:flex;gap:10px;justify-content:center}
.cdg b{width:58px;height:72px;border-radius:16px;display:grid;place-items:center;font:700 34px ui-monospace,SFMono-Regular,Menlo,monospace;background:var(--s2);color:var(--ac);box-shadow:inset 0 0 0 1.5px color-mix(in srgb,var(--ac) 40%,transparent)}
.cr .tile{background:var(--s1);border-radius:16px;padding:13px;display:flex;flex-direction:column;gap:6px;align-items:flex-start;text-align:left;box-shadow:inset 0 0 0 1px var(--ln);transition:transform .12s,background .15s}
.cr .tile:active{transform:scale(.97)}.cr .tile:hover{background:var(--s2)}
.tilegrid{display:grid;grid-template-columns:1fr 1fr;gap:10px}

/* ===== JOVEM+ ===== */
.jv .top{background:linear-gradient(160deg,#0B2447 0%,#123B7A 60%,#1F6FEB 130%);color:#fff;padding:16px 18px 74px;border-radius:0 0 30px 30px;position:relative}
.jv .topbar{display:flex;align-items:center;justify-content:space-between}
.jv .iconbtn{width:40px;height:40px;border-radius:13px;background:#ffffff1f;display:grid;place-items:center;position:relative;color:#fff}
.jv .dot{position:absolute;top:8px;right:9px;width:9px;height:9px;border-radius:50%;background:#19D98B;box-shadow:0 0 0 2px #123B7A}
.jv .nav{display:flex;background:var(--s1);border-top:1px solid var(--ln);padding:6px 6px 10px;gap:2px}
.jv .nav button{flex:1;display:flex;flex-direction:column;align-items:center;gap:3px;padding:8px 0;border-radius:14px;font-size:11px;font-weight:700;color:var(--mut);transition:color .15s,background .15s}
.jv .nav button.on{color:var(--ac);background:color-mix(in srgb,var(--ac) 11%,transparent)}
.jv .float{margin:-56px 18px 0;position:relative}
.jv .tile{background:var(--s1);border-radius:16px;padding:12px 6px;display:flex;flex-direction:column;align-items:center;gap:7px;font-size:12px;font-weight:700;box-shadow:var(--sh);transition:transform .12s}
.jv .tile:active{transform:scale(.95)}
.jv .tile .ti{width:42px;height:42px;border-radius:14px;display:grid;place-items:center;background:color-mix(in srgb,var(--ac) 12%,transparent);color:var(--ac)}
.seg{display:flex;background:var(--s2);border-radius:14px;padding:4px;gap:2px}
.seg button{flex:1;padding:9px 4px;border-radius:11px;font-size:12.5px;font-weight:700;color:var(--mut);transition:all .15s}
.seg button.on{background:var(--s1);color:var(--tx);box-shadow:0 2px 8px #0b244722}
.bar{height:10px;border-radius:99px;background:var(--s2);overflow:hidden}.bar>i{display:block;height:100%;border-radius:99px;background:linear-gradient(90deg,#1F6FEB,#19C37D);transition:width .6s cubic-bezier(.2,.9,.3,1)}
.pcard{position:relative;border-radius:22px;padding:18px;color:#fff;min-height:176px;display:flex;flex-direction:column;justify-content:space-between;overflow:hidden;box-shadow:0 16px 30px #0b24474d;transition:filter .3s,transform .3s}
.pcard.blk{filter:grayscale(1) brightness(.75)}
.pcard .wm{position:absolute;right:-8px;bottom:-14px;font-size:44px;font-weight:900;letter-spacing:.06em;opacity:.12;transform:rotate(-8deg)}
.pcard.debit{background:linear-gradient(135deg,#0B2447,#1F6FEB)}.pcard.credit{background:linear-gradient(135deg,#0E7C55,#19C37D)}.pcard.virtual{background:linear-gradient(135deg,#1B2B4B,#5B7BFF 70%,#19C37D)}.pcard.phys{background:#C9D3E3;color:#5C6F8A;box-shadow:none}
.chipic{width:38px;height:28px;border-radius:7px;background:linear-gradient(135deg,#f6dc8c,#c79b36);opacity:.95}
.recpt{position:relative;background:#fff;color:#0B2447;border-radius:18px;padding:18px;overflow:hidden;box-shadow:0 0 0 1px #0b244722}
.recpt .wm{position:absolute;inset:0;display:grid;place-items:center;font-size:46px;font-weight:900;color:#d9363e;opacity:.09;transform:rotate(-24deg);pointer-events:none;letter-spacing:.08em}
.acc{border-radius:16px;background:var(--s1);box-shadow:var(--sh);overflow:hidden;margin-top:10px}
.acc>button{width:100%;display:flex;align-items:center;gap:12px;padding:15px;text-align:left;font-weight:700}
.acc .body{padding:0 15px 15px;font-size:14px;line-height:1.55;color:var(--mut)}
.qr{background:#fff;border-radius:16px;padding:12px;width:208px;margin:0 auto;position:relative}
.pin{width:46px;height:46px;border-radius:50%;position:absolute;border:2px solid #19C37D;animation:ring 1.6s infinite}
.logo-wrap{animation:bob 4s ease-in-out infinite}
.tog{width:46px;height:27px;border-radius:99px;background:var(--s2);position:relative;transition:background .2s;flex:none}
.tog::after{content:"";position:absolute;top:3px;left:3px;width:21px;height:21px;border-radius:50%;background:#fff;box-shadow:0 1px 4px #0004;transition:transform .2s}
.tog.on{background:var(--ac)}.tog.on::after{transform:translateX(19px)}
.cdgin{letter-spacing:.7em;text-align:center;font:700 30px ui-monospace,SFMono-Regular,Menlo,monospace;padding-left:.7em}
.stepper{display:flex;gap:6px;margin:6px 0 4px}.stepper i{flex:1;height:5px;border-radius:9px;background:var(--s2)}.stepper i.on{background:var(--ac)}
`;

/* ---------- utilidades de UI ---------- */
const copyText = async (text) => {
  try {
    await navigator.clipboard.writeText(text);
    return true;
  } catch (e) {
    try {
      const ta = document.createElement("textarea");
      ta.value = text;
      ta.style.position = "fixed";
      ta.style.opacity = "0";
      document.body.appendChild(ta);
      ta.select();
      const ok = document.execCommand("copy");
      document.body.removeChild(ta);
      return ok;
    } catch (e2) {
      return false;
    }
  }
};
const wait = (ms) => new Promise((r) => setTimeout(r, ms));
function useNow(active) {
  const [now, setNow] = useState(Date.now());
  useEffect(() => {
    setNow(Date.now());
    if (!active) return;
    const t = setInterval(() => setNow(Date.now()), 500);
    return () => clearInterval(t);
  }, [active]);
  return now;
}
const maskIdInput = (v) => {
  const s = v.toUpperCase().replace(/[^A-Z0-9]/g, "").slice(0, 11);
  return s.slice(0, 3) + (s.length > 3 ? "-" + s.slice(3, 7) : "") + (s.length > 7 ? "-" + s.slice(7) : "");
};
const maskPhoneInput = (v) => {
  let d = digitsOf(v);
  if (d.length > 11 && d.startsWith("55")) d = d.slice(2);
  d = d.slice(0, 11);
  let o = "";
  if (d.length > 0) o = "(" + d.slice(0, 2);
  if (d.length >= 2) o += ") ";
  if (d.length > 2) o += d.slice(2, 7);
  if (d.length > 7) o += "-" + d.slice(7);
  return o;
};
const ID_RE = /^CRT-[A-Z0-9]{4}-[A-Z0-9]{4}$/;
const EMAIL_RE = /^[a-z0-9]+@jvpl\.demo$/i;
const NAME_RE = /^[A-Za-z\u00C0-\u00FF][A-Za-z\u00C0-\u00FF'\u2019\- ]{2,39}$/;

/* ---------- logotipos ---------- */
function CrLogo({ size = 56 }) {
  return (
    <svg width={size} height={size} viewBox="0 0 64 64" aria-label="CRTPL">
      <defs>
        <linearGradient id="crg" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stopColor="#FFD97A" /><stop offset="1" stopColor="#F29B3C" /></linearGradient>
      </defs>
      <rect x="4" y="10" width="56" height="44" rx="13" fill="#2A2147" stroke="url(#crg)" strokeWidth="2.5" />
      <path d="M4 24h56" stroke="url(#crg)" strokeWidth="2.5" opacity=".55" />
      <path d="M40 31a10 10 0 1 0 0 14" fill="none" stroke="url(#crg)" strokeWidth="5" strokeLinecap="round" />
      <circle cx="46" cy="38" r="3.2" fill="#FFC24B" />
    </svg>
  );
}
function JvLogo({ size = 56, light }) {
  return (
    <svg width={size} height={size} viewBox="0 0 64 64" aria-label="JOVEM+">
      <defs>
        <linearGradient id="jvg" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stopColor="#1F6FEB" /><stop offset="1" stopColor="#19C37D" /></linearGradient>
      </defs>
      <rect x="4" y="4" width="56" height="56" rx="18" fill={light ? "#fff" : "url(#jvg)"} />
      <path d="M20 40l9-9 6 6 11-13" fill="none" stroke={light ? "#1F6FEB" : "#fff"} strokeWidth="5" strokeLinecap="round" strokeLinejoin="round" />
      <path d="M40 24h7v7" fill="none" stroke={light ? "#19C37D" : "#fff"} strokeWidth="5" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  );
}

/* ---------- componentes compartilhados ---------- */
function Field({ label, value, onChange, type = "text", placeholder, error, hint, secret, inputMode, maxLength, onEnter, autoFocus, className = "", disabled }) {
  const [show, setShow] = useState(false);
  return (
    <div className="field">
      {label && <label>{label}</label>}
      <div className="inw">
        <input
          className={`inp ${error ? "bad" : ""} ${secret ? "pad" : ""} ${className}`}
          type={secret && !show ? "password" : type}
          value={value}
          placeholder={placeholder}
          inputMode={inputMode}
          maxLength={maxLength}
          autoFocus={autoFocus}
          disabled={disabled}
          autoComplete="off"
          autoCapitalize="none"
          spellCheck={false}
          onChange={(e) => onChange(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && onEnter && onEnter()}
        />
        {secret && (
          <button className="eye" type="button" aria-label={show ? "Ocultar senha" : "Mostrar senha"} onClick={() => setShow(!show)}>
            {show ? <EyeOff size={19} /> : <Eye size={19} />}
          </button>
        )}
      </div>
      {error ? <div className="err"><AlertTriangle size={15} style={{ flex: "none", marginTop: 1 }} />{error}</div> : hint ? <div className="xs mut">{hint}</div> : null}
    </div>
  );
}
function Btn({ children, onClick, kind = "", loading, disabled, icon: Icon, small }) {
  return (
    <button className={`btn ${kind} ${small ? "sm2" : ""}`} onClick={onClick} disabled={disabled || loading}>
      {loading ? <Loader2 size={18} className="spin" /> : Icon ? <Icon size={18} /> : null}
      {children}
    </button>
  );
}
function Back({ onClick, label = "Voltar" }) {
  return <button className="back" onClick={onClick}><ArrowLeft size={18} />{label}</button>;
}
function Sheet({ onClose, title, children }) {
  return (
    <div className="ovl" onClick={onClose}>
      <div className="sheet" onClick={(e) => e.stopPropagation()}>
        <div className="grab" />
        <div className="row sb" style={{ marginBottom: 6 }}>
          <h2>{title}</h2>
          <button className="eye" style={{ position: "static" }} aria-label="Fechar" onClick={onClose}><X size={20} /></button>
        </div>
        {children}
      </div>
    </div>
  );
}
function Confirm({ title, text, confirmLabel = "Confirmar", onConfirm, onCancel, danger, loading }) {
  return (
    <div className="ovl" style={{ alignItems: "center" }} onClick={onCancel}>
      <div className="dlg" onClick={(e) => e.stopPropagation()} style={{ width: "calc(100% - 36px)" }}>
        <h2>{title}</h2>
        <div className="mut mt8" style={{ fontSize: 14.5, lineHeight: 1.5 }}>{text}</div>
        <div className="row g8 mt16">
          <Btn kind="sec" onClick={onCancel} disabled={loading}>Cancelar</Btn>
          <Btn kind={danger ? "danger" : ""} onClick={onConfirm} loading={loading}>{confirmLabel}</Btn>
        </div>
      </div>
    </div>
  );
}
function MoneyInput({ cents, onChange, label = "Valor", error }) {
  const [txt, setTxt] = useState(cents ? String(cents) : "");
  useEffect(() => { if (!cents) setTxt(""); }, [cents]);
  return (
    <Field
      label={label}
      value={cents ? brl(cents) : ""}
      placeholder="R$ 0,00"
      inputMode="numeric"
      error={error}
      onChange={(v) => {
        const d = digitsOf(v).replace(/^0+/, "").slice(0, 9);
        setTxt(d);
        onChange(d ? parseInt(d, 10) : 0);
      }}
    />
  );
}
function Empty({ icon: Icon, title, text }) {
  return (
    <div className="col ctr" style={{ alignItems: "center", padding: "30px 16px", gap: 8 }}>
      <div className="ico" style={{ width: 56, height: 56, borderRadius: 18 }}><Icon size={26} className="mut" /></div>
      <h3>{title}</h3>
      <p className="mut sm" style={{ maxWidth: 260 }}>{text}</p>
    </div>
  );
}
function CopyBtn({ text, label, toast, kind = "sec", small = true }) {
  const [ok, setOk] = useState(false);
  return (
    <Btn
      kind={kind}
      small={small}
      icon={ok ? Check : Copy}
      onClick={async () => {
        const r = await copyText(text);
        if (r) { setOk(true); toast("Copiado!", "ok"); setTimeout(() => setOk(false), 1400); }
        else toast("Não foi possível copiar automaticamente. Selecione o texto.", "er");
      }}
    >{label}</Btn>
  );
}
function LockBanner({ remaining }) {
  return (
    <div className="banner er" role="alert">
      <Clock size={18} />
      <div><b>{MSG_LOCK}</b><div className="mt8 b" style={{ fontSize: 22, fontFamily: "ui-monospace,Menlo,monospace" }}>{mmss(remaining)}</div><div className="xs mut">Tempo restante do bloqueio. Depois dele, você terá mais {MAX_TRIES} tentativas.</div></div>
    </div>
  );
}
function SimBanner({ children }) {
  return <div className="banner"><Info size={17} /><div>{children}</div></div>;
}
function Receipt_({ tx, balanceAfter, onClose, who }) {
  return (
    <Sheet title="Comprovante demonstrativo" onClose={onClose}>
      <div className="recpt">
        <div className="wm">SIMULAÇÃO</div>
        <div className="row g8"><JvLogo size={30} /><div><b>JOVEM+</b><div className="xs" style={{ color: "#5C6F8A" }}>{fmtDate(tx.date)}</div></div>
          <span className="badge er" style={{ marginLeft: "auto" }}>SIMULAÇÃO</span></div>
        <div className="sep" />
        <div style={{ color: "#5C6F8A", fontSize: 13 }}>{tx.title}</div>
        <div className="b" style={{ fontSize: 30, letterSpacing: "-.02em" }}>{brl(tx.amount)}</div>
        {[
          [tx.type === "out" ? "Para" : "De", tx.counter.name],
          ["Identificador", tx.counter.idMasked],
          who && [tx.type === "out" ? "De" : "Para", who],
          tx.desc && ["Descrição", tx.desc],
          balanceAfter != null && ["Saldo restante", brl(balanceAfter)],
          ["Protocolo", tx.receipt],
        ].filter(Boolean).map(([k, v]) => (
          <div key={k} className="row sb" style={{ padding: "8px 0", borderBottom: "1px dashed #0b244722", gap: 12 }}>
            <span style={{ color: "#5C6F8A", fontSize: 13 }}>{k}</span><b style={{ fontSize: 13.5, textAlign: "right", wordBreak: "break-all" }}>{v}</b>
          </div>
        ))}
        <p className="xs" style={{ color: "#5C6F8A", marginTop: 12 }}>Comprovante fictício. Nenhum dinheiro real foi movimentado e nenhum sistema Pix real foi acionado.</p>
      </div>
      <div className="mt16"><Btn onClick={onClose}>Concluir</Btn></div>
    </Sheet>
  );
}
function FakeQR({ seed }) {
  const N = 25;
  const cells = useMemo(() => {
    let h = 7;
    for (const c of seed) h = Math.imul(h ^ c.charCodeAt(0), 16777619);
    const rng = () => ((h = Math.imul(h ^ (h >>> 15), 2246822507) ^ Math.imul(h ^ (h >>> 13), 3266489909)) >>> 0) / 4294967296;
    const out = [];
    const finder = (x, y) => (x < 8 && y < 8) || (x >= N - 8 && y < 8) || (x < 8 && y >= N - 8);
    for (let y = 0; y < N; y++) for (let x = 0; x < N; x++) if (!finder(x, y) && rng() > 0.52) out.push([x, y]);
    return out;
  }, [seed]);
  const F = ({ x, y }) => (
    <g transform={`translate(${x},${y})`}><rect width="7" height="7" fill="#0B2447" /><rect x="1" y="1" width="5" height="5" fill="#fff" /><rect x="2" y="2" width="3" height="3" fill="#0B2447" /></g>
  );
  return (
    <div className="qr">
      <svg viewBox={`0 0 ${N} ${N}`} width="100%" shapeRendering="crispEdges">
        {cells.map(([x, y]) => <rect key={x + "-" + y} x={x} y={y} width="1" height="1" fill="#0B2447" />)}
        <F x={0} y={0} /><F x={N - 7} y={0} /><F x={0} y={N - 7} />
      </svg>
      <div style={{ position: "absolute", inset: 0, display: "grid", placeItems: "center", fontWeight: 900, fontSize: 30, color: "#D9363E", opacity: 0.22, transform: "rotate(-24deg)", letterSpacing: ".05em" }}>SIMULAÇÃO</div>
    </div>
  );
}
function useLock() {
  const [until, setUntil] = useState(0);
  const now = useNow(until > Date.now());
  const rem = until - now;
  return { rem: rem > 0 ? rem : 0, lock: (ms) => setUntil(Date.now() + ms), clear: () => setUntil(0) };
}

function Guilloche({ size = 220 }) {
  return (
    <svg className="guil" width={size} height={size} viewBox="0 0 200 200" fill="none" stroke="#150F26" strokeWidth=".8">
      {Array.from({ length: 14 }, (_, i) => <ellipse key={i} cx="100" cy="100" rx={90 - i * 3} ry={30 + i * 4} transform={`rotate(${i * 13} 100 100)`} />)}
    </svg>
  );
}

function IdentityCard({ ident, reveal }) {
  return (
    <div className="idcard">
      <Guilloche />
      <div className="row sb" style={{ position: "relative" }}>
        <div className="row g8"><CrLogo size={34} /><div><div className="b" style={{ fontSize: 15 }}>CRTPL</div><div className="lab">CARTEIRA PESSOAL</div></div></div>
        <span className="badge" style={{ background: "#150F2622", color: "#150F26" }}><Fingerprint size={13} />{ident.status}</span>
      </div>
      <div style={{ marginTop: 26, position: "relative" }}>
        <div className="lab">NOME</div>
        <div className="val" style={{ fontSize: 21, letterSpacing: "-.01em" }}>{ident.name}</div>
      </div>
      <div className="row sb" style={{ marginTop: 14, position: "relative", gap: 10 }}>
        <div className="grow"><div className="lab">IDENTIFICADOR (FICTÍCIO)</div><div className="val" style={{ fontFamily: "ui-monospace,Menlo,monospace" }}>{reveal ? ident.identifier : maskIdent(ident.identifier)}</div></div>
        <div><div className="lab">CDG</div><div className="val" style={{ fontFamily: "ui-monospace,Menlo,monospace", fontSize: 18 }}>{reveal ? ident.cdg : "••••"}</div></div>
      </div>
      <div style={{ marginTop: 12, position: "relative" }}>
        <div className="lab">E-MAIL JVPL</div><div className="val" style={{ fontSize: 14 }}>{ident.email}</div>
      </div>
      <div className="stamp">DEMO · NÃO OFICIAL</div>
    </div>
  );
}

function CrWelcome({ go, exit }) {
  return (
    <div className="scroll" style={{ padding: 0 }}>
      <div className="hero" style={{ minHeight: "100%", display: "flex", flexDirection: "column" }}>
        <div className="glow" style={{ background: "#8F7BFF", top: -80, left: -90 }} />
        <div className="glow" style={{ background: "#FFC24B", bottom: 40, right: -120, opacity: 0.22 }} />
        <button className="back" onClick={exit} style={{ position: "relative" }}><ArrowLeft size={18} />Trocar de aplicativo</button>
        <div className="col rise" style={{ alignItems: "flex-start", marginTop: 34, position: "relative", gap: 16 }}>
          <div className="logo-wrap"><CrLogo size={84} /></div>
          <div>
            <h1 style={{ fontSize: 44, letterSpacing: "-.04em" }}>CRTPL</h1>
            <div className="b" style={{ color: "var(--ac)", marginTop: 2 }}>Carteira Pessoal</div>
          </div>
          <p className="mut" style={{ lineHeight: 1.55, maxWidth: 330 }}>
            Crie uma identidade fictícia com e-mail JVPL, telefone demonstrativo e um CDG de 4 dígitos. É ela que abre as portas do JOVEM+.
          </p>
        </div>
        <div className="grow" />
        <div className="col g12 rise" style={{ position: "relative", marginTop: 28, animationDelay: ".15s" }}>
          <Btn icon={Fingerprint} onClick={() => go("create")}>Criar identidade</Btn>
          <Btn kind="sec" icon={Lock} onClick={() => go("login")}>Entrar</Btn>
          <button className="link" style={{ alignSelf: "center", padding: 6 }} onClick={() => go("forgot")}>Esqueci minha senha</button>
          <SimBanner>Projeto de demonstração. Nada aqui é documento oficial, e-mail, telefone ou conta real.</SimBanner>
        </div>
      </div>
    </div>
  );
}

function CrCreate({ db, commit, go, toast, setCrUser }) {
  const [name, setName] = useState("");
  const [p1, setP1] = useState("");
  const [p2, setP2] = useState("");
  const [loading, setLoading] = useState(false);
  const [confirm, setConfirm] = useState(false);
  const [created, setCreated] = useState(null);
  const [errs, setErrs] = useState({});
  const nn = normName(name);
  const dupLive = nn.length >= 3 && (db.identities.some((i) => i.normName === nn || i.email === slug(name) + "@jvpl.demo") || SEEDS.some((s) => normName(s.name) === nn));

  const validate = () => {
    const e = {};
    const nm = cleanName(name);
    if (!NAME_RE.test(nm) || !slug(nm)) e.name = "Use de 3 a 40 letras (espaços, hífen e apóstrofo são aceitos).";
    else if (dupLive) e.name = DUP_MSG;
    if (p1.length < 6) e.p1 = "A senha precisa ter pelo menos 6 caracteres.";
    if (p2 !== p1) e.p2 = "A confirmação não é igual à senha.";
    setErrs(e);
    return Object.keys(e).length === 0;
  };
  const create = async () => {
    setLoading(true);
    await wait(1100);
    const salt = newSalt();
    const hash = await hashPass(p1, salt);
    const r = commit((d) => createIdentity(d, { name, passHash: hash, salt }, Date.now()));
    setLoading(false);
    setConfirm(false);
    if (r.error) { setErrs({ name: r.error }); return; }
    setCreated(r.identity);
  };

  if (created)
    return (
      <div className="scroll">
        <div className="col rise" style={{ alignItems: "center", gap: 6, textAlign: "center", marginTop: 4 }}>
          <div className="ico" style={{ background: "color-mix(in srgb,var(--ok) 20%,transparent)", width: 56, height: 56, borderRadius: 18 }}><Check size={28} color="var(--ok)" /></div>
          <h2>Identidade criada!</h2>
          <p className="mut sm">Estes dados foram gerados e salvos na base demonstrativa.</p>
        </div>
        <div className="mt16"><IdentityCard ident={created} reveal /></div>
        <div className="card mt16">
          <div className="irow"><Smartphone size={18} className="mut" /><div className="grow"><div className="xs mut">Telefone demonstrativo</div><div className="b">{fmtPhone(created.phone)}</div></div></div>
          <div className="irow"><Landmark size={18} className="mut" /><div className="grow"><div className="xs mut">DDD · Operadora ilustrativa</div><div className="b">{created.ddd} · {created.carrier}</div></div></div>
          <div className="mt12 col" style={{ gap: 8 }}>
            <div className="xs mut" style={{ textAlign: "center" }}>Seu CDG — Código de Garantia</div>
            <div className="cdg">{created.cdg.split("").map((c, i) => <b key={i}>{c}</b>)}</div>
          </div>
        </div>
        <div className="mt12"><SimBanner>Anote o CDG: ele será pedido no JOVEM+. Ele não substitui a senha e não é autenticação bancária segura.</SimBanner></div>
        <div className="mt16"><Btn icon={Wallet} onClick={() => setCrUser(created.id)}>Abrir minha carteira</Btn></div>
      </div>
    );

  return (
    <div className="scroll">
      <Back onClick={() => go("welcome")} />
      <h1>Criar identidade</h1>
      <p className="mut mt8" style={{ lineHeight: 1.5 }}>Escolha um nome fictício e uma senha. O resto é gerado por nós.</p>
      <Field label="Nome completo" value={name} onChange={(v) => { setName(v); setErrs({}); }} placeholder="Ex.: Cleitindo Grau" error={errs.name || (dupLive ? DUP_MSG : "")} hint={nn.length >= 3 && !dupLive ? undefined : "Letras, espaços e hífen. Ignoramos maiúsculas e espaços extras na comparação."} maxLength={40} />
      {nn.length >= 3 && !dupLive && NAME_RE.test(cleanName(name)) && <div className="okmsg mt8"><Check size={15} />Nome disponível. E-mail previsto: {slug(name)}@jvpl.demo</div>}
      <Field label="Senha" secret value={p1} onChange={(v) => { setP1(v); setErrs({}); }} placeholder="Mínimo de 6 caracteres" error={errs.p1} />
      <Field label="Confirmar senha" secret value={p2} onChange={(v) => { setP2(v); setErrs({}); }} placeholder="Repita a senha" error={errs.p2} onEnter={() => validate() && setConfirm(true)} />
      <div className="mt24"><Btn icon={Sparkles} onClick={() => validate() && setConfirm(true)}>CRIAR MINHA IDENTIDADE</Btn></div>
      <div className="mt12"><SimBanner>Use apenas informações fictícias. Não digite senhas reais de banco, e-mail ou redes sociais.</SimBanner></div>
      {confirm && (
        <Confirm title="Criar esta identidade?" text={<>Vamos gerar identificador, e-mail JVPL, telefone demonstrativo e CDG para <b>{cleanName(name)}</b>.</>} confirmLabel="Criar agora" onConfirm={create} onCancel={() => setConfirm(false)} loading={loading} />
      )}
    </div>
  );
}

function CrLogin({ getDb, commit, go, setCrUser, toast }) {
  const [key, setKey] = useState("");
  const [pass, setPass] = useState("");
  const [err, setErr] = useState("");
  const [loading, setLoading] = useState(false);
  const [shake, setShake] = useState(0);
  const L = useLock();
  const submit = async () => {
    setErr("");
    if (!key.trim() || !pass) { setErr("Informe o nome ou identificador e a senha."); return; }
    setLoading(true);
    await wait(800);
    const d = getDb();
    const ident = findIdentity(d, key);
    if (!ident) { setLoading(false); setErr("Identidade não encontrada. Confira o nome, e-mail ou identificador."); setShake((s) => s + 1); return; }
    const bucket = "crlogin:" + ident.id;
    const li = lockInfo(d, bucket, Date.now());
    if (li.locked) { setLoading(false); L.lock(li.remaining); return; }
    const h = await hashPass(pass, ident.salt);
    if (h === ident.passHash) {
      commit((x) => ({ db: clearLock(x, bucket) }));
      setLoading(false);
      setCrUser(ident.id);
      return;
    }
    const r = commit((x) => failResult(x, bucket, Date.now()));
    setLoading(false);
    setShake((s) => s + 1);
    if (r.status === "locked") L.lock(r.remaining);
    else setErr(`Senha incorreta. Tentativas restantes: ${r.left}.`);
  };
  return (
    <div className="scroll">
      <Back onClick={() => go("welcome")} />
      <div className="row g12"><CrLogo size={44} /><h1>Entrar</h1></div>
      <p className="mut mt8">Use seu nome ou identificador CRTPL.</p>
      <div key={shake} className={shake ? "shake" : ""}>
        <Field label="Nome ou identificador" value={key} onChange={(v) => { setKey(v); setErr(""); }} placeholder="Nome, e-mail JVPL ou CRT-XXXX-XXXX" />
        <Field label="Senha" secret value={pass} onChange={(v) => { setPass(v); setErr(""); }} onEnter={submit} />
      </div>
      {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
      {L.rem > 0 && <div className="mt12"><LockBanner remaining={L.rem} /></div>}
      <div className="mt24"><Btn onClick={submit} loading={loading} disabled={L.rem > 0} icon={Lock}>Entrar</Btn></div>
      <div className="row sb mt16">
        <button className="link" onClick={() => go("forgot")}>Esqueci minha senha</button>
        <button className="link" onClick={() => go("create")}>Criar identidade</button>
      </div>
    </div>
  );
}

function CrForgot({ getDb, commit, go, toast }) {
  const [step, setStep] = useState(0);
  const [key, setKey] = useState("");
  const [cdg, setCdg] = useState("");
  const [p1, setP1] = useState("");
  const [p2, setP2] = useState("");
  const [err, setErr] = useState("");
  const [loading, setLoading] = useState(false);
  const [identId, setIdentId] = useState(null);
  const L = useLock();
  useEffect(() => { const li = lockInfo(getDb(), "crrecover", Date.now()); if (li.locked) L.lock(li.remaining); }, []);
  const verify = async () => {
    setErr("");
    if (!key.trim() || cdg.length !== 4) { setErr("Informe o nome (ou identificador) e o CDG de 4 dígitos."); return; }
    setLoading(true);
    await wait(900);
    const r = commit((d) => verifyCDG(d, findIdentity(d, key), cdg, Date.now(), "crrecover"));
    setLoading(false);
    if (r.status === "ok") { setIdentId(r.identity.id); setStep(1); }
    else if (r.status === "locked") L.lock(r.remaining);
    else setErr(`${MSG_FAIL} Tentativas restantes: ${r.left}.`);
  };
  const save = async () => {
    if (p1.length < 6) { setErr("A senha precisa ter pelo menos 6 caracteres."); return; }
    if (p1 !== p2) { setErr("A confirmação não é igual à senha."); return; }
    setLoading(true);
    const salt = newSalt();
    const hash = await hashPass(p1, salt);
    await wait(700);
    commit((d) => ({ db: { ...d, identities: d.identities.map((i) => (i.id === identId ? { ...i, passHash: hash, salt } : i)) } }));
    setLoading(false);
    toast("Senha redefinida. Entre com a nova senha.", "ok");
    go("login");
  };
  return (
    <div className="scroll">
      <Back onClick={() => go("welcome")} />
      <h1>Recuperar senha</h1>
      <p className="mut mt8" style={{ lineHeight: 1.5 }}>Recuperação demonstrativa: nenhum SMS ou e-mail é enviado. Confirme sua identidade com o CDG.</p>
      <div className="stepper mt12"><i className="on" /><i className={step ? "on" : ""} /></div>
      {step === 0 ? (
        <>
          <Field label="Nome ou identificador" value={key} onChange={(v) => { setKey(v); setErr(""); }} placeholder="Nome, e-mail JVPL ou CRT-XXXX-XXXX" />
          <Field label="CDG (4 dígitos)" value={cdg} inputMode="numeric" maxLength={4} onChange={(v) => { setCdg(digitsOf(v).slice(0, 4)); setErr(""); }} placeholder="0000" onEnter={verify} />
          {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
          {L.rem > 0 && <div className="mt12"><LockBanner remaining={L.rem} /></div>}
          <div className="mt24"><Btn onClick={verify} loading={loading} disabled={L.rem > 0} icon={ShieldCheck}>Confirmar identidade</Btn></div>
        </>
      ) : (
        <>
          <div className="banner ok mt12"><Check size={17} /><div>Identidade confirmada. Defina uma nova senha fictícia.</div></div>
          <Field label="Nova senha" secret value={p1} onChange={(v) => { setP1(v); setErr(""); }} />
          <Field label="Confirmar nova senha" secret value={p2} onChange={(v) => { setP2(v); setErr(""); }} onEnter={save} />
          {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
          <div className="mt24"><Btn onClick={save} loading={loading} icon={Lock}>Salvar nova senha</Btn></div>
        </>
      )}
    </div>
  );
}

function CrHome({ db, commit, getDb, ident, toast, onLogout, goJovem }) {
  const [sheet, setSheet] = useState(null);
  const [confirmOut, setConfirmOut] = useState(false);
  const acct = accountOf(db, ident.id);
  const doCopy = async (txt, what) => ((await copyText(txt)) ? toast(`${what} copiado!`, "ok") : toast("Não foi possível copiar. Selecione o texto manualmente.", "er"));
  const Row = ({ icon: I, k, v, mono }) => (
    <div className="irow"><I size={18} className="mut" /><div className="grow"><div className="xs mut">{k}</div><div className="b ell" style={mono ? { fontFamily: "ui-monospace,Menlo,monospace" } : {}}>{v}</div></div></div>
  );
  return (
    <div className="scroll">
      <div className="row sb">
        <div className="row g12">
          <CrLogo size={38} />
          <div><div className="xs mut">CRTPL · Carteira Pessoal</div><h2 style={{ fontSize: 19 }}>Bem-vindo(a), {firstName(ident.name)}!</h2></div>
        </div>
      </div>
      <div className="mt16"><IdentityCard ident={ident} /></div>

      <div className="card mt16">
        <div className="xs mut" style={{ textAlign: "center" }}>CDG — Código de Garantia</div>
        <div className="cdg mt8">{ident.cdg.split("").map((c, i) => <b key={i}>{c}</b>)}</div>
        <p className="xs mut mt12" style={{ textAlign: "center" }}>Usado para verificar sua identidade no JOVEM+. Não é senha nem autenticação bancária.</p>
      </div>

      <div className="card mt12">
        <Row icon={User} k="Nome" v={ident.name} />
        <Row icon={Fingerprint} k="Identificador pessoal (mascarado)" v={maskIdent(ident.identifier)} mono />
        <Row icon={Mail} k="E-mail JVPL (fictício)" v={ident.email} />
        <Row icon={Smartphone} k="Telefone demonstrativo" v={fmtPhone(ident.phone)} />
        <Row icon={Landmark} k="DDD · Operadora ilustrativa" v={`${ident.ddd} · ${ident.carrier}`} />
        <Row icon={Clock} k="Criada em" v={fmtDate(ident.createdAt)} />
        <div className="irow"><ShieldCheck size={18} className="mut" /><div className="grow"><div className="xs mut">Status da identidade</div><span className="badge">{ident.status}</span></div></div>
        <div className="irow"><Link2 size={18} className="mut" /><div className="grow"><div className="xs mut">Conta JOVEM+ vinculada</div><div className="b">{acct ? `Sim · criada em ${fmtDay(acct.createdAt)}` : "Nenhuma ainda"}</div></div></div>
      </div>

      <div className="tilegrid mt16">
        <button className="tile" onClick={() => doCopy(ident.email, "E-mail")}><Mail size={20} color="var(--ac)" /><b className="sm">Copiar e-mail</b></button>
        <button className="tile" onClick={() => doCopy(fmtPhone(ident.phone), "Número")}><Smartphone size={20} color="var(--ac)" /><b className="sm">Copiar número</b></button>
        <button className="tile" onClick={() => doCopy(ident.cdg, "CDG")}><Copy size={20} color="var(--ac)" /><b className="sm">Copiar CDG</b></button>
        <button className="tile" onClick={() => setSheet("view")}><Eye size={20} color="var(--ac)" /><b className="sm">Visualizar identidade</b></button>
        <button className="tile" onClick={() => setSheet("pass")}><Lock size={20} color="var(--ac)" /><b className="sm">Alterar senha</b></button>
        <button className="tile" onClick={() => setConfirmOut(true)}><LogOut size={20} color="var(--er)" /><b className="sm">Sair</b></button>
      </div>
      <div className="mt16"><Btn kind="out" icon={Landmark} onClick={goJovem}>Abrir o JOVEM+</Btn></div>
      <div className="mt12"><SimBanner>Todos os dados desta tela são fictícios. O e-mail .demo e o telefone não recebem mensagens nem chamadas reais.</SimBanner></div>

      {sheet === "view" && <ViewIdentity ident={ident} onClose={() => setSheet(null)} doCopy={doCopy} />}
      {sheet === "pass" && <CrChangePass ident={ident} commit={commit} onClose={() => setSheet(null)} toast={toast} />}
      {confirmOut && <Confirm title="Sair do CRTPL?" text="Você precisará entrar novamente com nome e senha." confirmLabel="Sair" onConfirm={onLogout} onCancel={() => setConfirmOut(false)} />}
    </div>
  );
}

function ViewIdentity({ ident, onClose, doCopy }) {
  const [reveal, setReveal] = useState(false);
  return (
    <Sheet title="Minha identidade" onClose={onClose}>
      <IdentityCard ident={ident} reveal={reveal} />
      <div className="card mt12">
        <div className="irow"><div className="grow"><div className="xs mut">Identificador completo (fictício)</div><div className="b" style={{ fontFamily: "ui-monospace,Menlo,monospace" }}>{reveal ? ident.identifier : maskIdent(ident.identifier)}</div></div>
          {reveal && <button className="eye" style={{ position: "static" }} aria-label="Copiar identificador" onClick={() => doCopy(ident.identifier, "Identificador")}><Copy size={18} /></button>}</div>
        <div className="irow"><div className="grow"><div className="xs mut">Telefone</div><div className="b">{fmtPhone(ident.phone)}</div></div></div>
        <div className="irow"><div className="grow"><div className="xs mut">Data de criação</div><div className="b">{fmtDate(ident.createdAt)}</div></div></div>
      </div>
      <div className="row g8 mt16">
        <Btn kind="sec" icon={reveal ? EyeOff : Eye} onClick={() => setReveal(!reveal)}>{reveal ? "Ocultar dados" : "Mostrar dados"}</Btn>
        <Btn onClick={onClose}>Fechar</Btn>
      </div>
      <p className="xs mut mt12" style={{ textAlign: "center" }}>Identidade fictícia, sem validade como documento oficial.</p>
    </Sheet>
  );
}

function CrChangePass({ ident, commit, onClose, toast }) {
  const [cur, setCur] = useState("");
  const [p1, setP1] = useState("");
  const [p2, setP2] = useState("");
  const [err, setErr] = useState("");
  const [loading, setLoading] = useState(false);
  const save = async () => {
    setErr("");
    if (p1.length < 6) return setErr("A nova senha precisa ter pelo menos 6 caracteres.");
    if (p1 !== p2) return setErr("A confirmação não é igual à nova senha.");
    setLoading(true);
    const h = await hashPass(cur, ident.salt);
    await wait(700);
    if (h !== ident.passHash) { setLoading(false); return setErr("Senha atual incorreta."); }
    const salt = newSalt();
    const nh = await hashPass(p1, salt);
    commit((d) => ({ db: { ...d, identities: d.identities.map((i) => (i.id === ident.id ? { ...i, passHash: nh, salt } : i)) } }));
    setLoading(false);
    toast("Senha alterada com sucesso.", "ok");
    onClose();
  };
  return (
    <Sheet title="Alterar senha" onClose={onClose}>
      <Field label="Senha atual" secret value={cur} onChange={(v) => { setCur(v); setErr(""); }} />
      <Field label="Nova senha" secret value={p1} onChange={(v) => { setP1(v); setErr(""); }} />
      <Field label="Confirmar nova senha" secret value={p2} onChange={(v) => { setP2(v); setErr(""); }} onEnter={save} />
      {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
      <div className="mt24"><Btn onClick={save} loading={loading} icon={Lock}>Salvar nova senha</Btn></div>
    </Sheet>
  );
}

function CrApp({ db, commit, getDb, toast, crUser, setCrUser, exit, goJovem }) {
  const [view, setView] = useState("welcome");
  const ident = crUser ? db.identities.find((i) => i.id === crUser) : null;
  useEffect(() => { if (crUser && !ident) setCrUser(null); }, [crUser, ident]);
  if (ident)
    return <CrHome db={db} commit={commit} getDb={getDb} ident={ident} toast={toast} goJovem={goJovem} onLogout={() => { setCrUser(null); setView("welcome"); toast("Você saiu do CRTPL.", "in"); }} />;
  if (view === "create") return <CrCreate db={db} commit={commit} go={setView} toast={toast} setCrUser={setCrUser} />;
  if (view === "login") return <CrLogin getDb={getDb} commit={commit} go={setView} setCrUser={setCrUser} toast={toast} />;
  if (view === "forgot") return <CrForgot getDb={getDb} commit={commit} go={setView} toast={toast} />;
  return <CrWelcome go={setView} exit={exit} />;
}
function JvWelcome({ go, exit }) {
  return (
    <div className="scroll" style={{ padding: 0, background: "linear-gradient(180deg,#0B2447 0%,#123B7A 55%,#F2F5FA 55.1%)" }}>
      <div style={{ padding: "20px 22px 28px", minHeight: "100%", display: "flex", flexDirection: "column" }}>
        <button className="back" onClick={exit} style={{ color: "#fff9" }}><ArrowLeft size={18} />Trocar de aplicativo</button>
        <div className="col rise" style={{ alignItems: "flex-start", gap: 14, marginTop: 20, color: "#fff" }}>
          <div className="logo-wrap"><JvLogo size={76} /></div>
          <div>
            <h1 style={{ fontSize: 42, letterSpacing: "-.04em" }}>JOVEM<span style={{ color: "#19D98B" }}>+</span></h1>
            <p style={{ fontSize: 18, marginTop: 4, opacity: 0.92 }}>Seu futuro começa agora.</p>
          </div>
        </div>
        <div className="card rise mt24" style={{ animationDelay: ".12s", padding: 20 }}>
          <div className="col g12">
            <div className="row g12"><div className="ico" style={{ background: "color-mix(in srgb,var(--ac) 12%,transparent)" }}><Fingerprint size={20} color="var(--ac)" /></div><div className="sm"><b>Entre com sua identidade CRTPL</b><div className="mut xs">Só entra quem tiver uma identidade válida.</div></div></div>
            <Btn onClick={() => go("login")} icon={Lock}>Entrar</Btn>
            <Btn kind="sec" onClick={() => go("register")} icon={Plus}>Criar conta</Btn>
            <button className="link" style={{ alignSelf: "center", padding: 4 }} onClick={() => go("forgot")}>Esqueci minha senha</button>
          </div>
        </div>
        <div className="grow" />
        <div className="mt16"><SimBanner>Banco demonstrativo. Nenhuma conta, cartão ou Pix aqui é real, e nenhum valor é movimentado.</SimBanner></div>
      </div>
    </div>
  );
}

function JvRegister({ db, getDb, commit, go, toast, setJvUser }) {
  const BUCKET = "jovem-verify";
  const [step, setStep] = useState(1);
  const [f, setF] = useState({ name: "", identifier: "", email: "", phone: "" });
  const [errs, setErrs] = useState({});
  const [cdg, setCdg] = useState("");
  const [msg, setMsg] = useState(null);
  const [loading, setLoading] = useState(false);
  const [proof, setProof] = useState(null);
  const [p1, setP1] = useState("");
  const [p2, setP2] = useState("");
  const [ack, setAck] = useState(false);
  const [shake, setShake] = useState(0);
  const L = useLock();
  useEffect(() => { const li = lockInfo(getDb(), BUCKET, Date.now()); if (li.locked) L.lock(li.remaining); }, []);
  const left = lockInfo(db, BUCKET, Date.now()).left;
  const set = (k, v) => { setF({ ...f, [k]: v }); setErrs({ ...errs, [k]: "" }); setMsg(null); };

  const validate1 = () => {
    const e = {};
    if (!NAME_RE.test(cleanName(f.name))) e.name = "Informe o nome completo da identidade.";
    if (!ID_RE.test(f.identifier.trim().toUpperCase())) e.identifier = "Formato: CRT-XXXX-XXXX.";
    if (!EMAIL_RE.test(f.email.trim())) e.email = "Formato: nome@jvpl.demo.";
    const ph = normPhone(f.phone);
    if (ph.length !== 11 || !ph.startsWith("67")) e.phone = "Formato: (67) 9XXXX-XXXX, com DDD 67.";
    setErrs(e);
    return !Object.keys(e).length;
  };
  const fail = (r) => {
    setShake((s) => s + 1);
    if (r.status === "locked") { L.lock(r.remaining); setMsg(null); }
    else setMsg({ t: "er", text: `${MSG_FAIL} Tentativas restantes: ${r.left}.` });
  };
  const step1 = async () => {
    if (!validate1()) return;
    setLoading(true);
    await wait(1400);
    const r = commit((d) => verifyData(d, f, Date.now(), BUCKET));
    setLoading(false);
    if (r.status === "ok") { setMsg(null); setStep(2); } else fail(r);
  };
  const step2 = async () => {
    if (cdg.length !== 4) { setMsg({ t: "er", text: "O CDG tem exatamente 4 dígitos." }); return; }
    setLoading(true);
    await wait(1400);
    const r = commit((d) => verifyIdentity(d, f, cdg, Date.now(), BUCKET));
    if (r.status !== "ok") { setLoading(false); setCdg(""); fail(r); return; }
    if (accountOf(getDb(), r.identity.id)) {
      setLoading(false);
      setMsg({ t: "er", text: "Esta identidade CRTPL já possui uma conta JOVEM+. Use Entrar." });
      return;
    }
    setProof(r.identity.id);
    setMsg({ t: "ok", text: MSG_OK });
    await wait(1300);
    setLoading(false);
    setMsg(null);
    setStep(3);
  };
  const step3 = async () => {
    setMsg(null);
    if (p1.length < 6) return setMsg({ t: "er", text: "A senha do JOVEM+ precisa ter pelo menos 6 caracteres." });
    if (p1 !== p2) return setMsg({ t: "er", text: "A confirmação não é igual à senha." });
    if (!ack) return setMsg({ t: "er", text: "Confirme que entende que este app é uma simulação." });
    setLoading(true);
    const ident = getDb().identities.find((i) => i.id === proof);
    if (!ident) { setLoading(false); return setMsg({ t: "er", text: MSG_FAIL }); }
    if ((await hashPass(p1, ident.salt)) === ident.passHash) { setLoading(false); return setMsg({ t: "er", text: "Use uma senha diferente da senha do CRTPL." }); }
    const salt = newSalt();
    const hash = await hashPass(p1, salt);
    await wait(1000);
    const r = commit((d) => createAccount(d, proof, hash, salt, Date.now()));
    setLoading(false);
    if (r.error) return setMsg({ t: "er", text: r.error });
    toast("Conta JOVEM+ criada!", "ok");
    setJvUser(r.account.id);
  };

  const Msg = () => msg && (
    <div className={`banner ${msg.t} mt12`} role="alert">{msg.t === "ok" ? <CircleCheck size={18} /> : <CircleX size={18} />}<div><b>{msg.text}</b></div></div>
  );
  return (
    <div className="scroll">
      <Back onClick={() => (step === 1 || step === 3 ? go("welcome") : setStep(1))} />
      <h1>Criar conta</h1>
      <div className="stepper mt12"><i className="on" /><i className={step >= 2 ? "on" : ""} /><i className={step >= 3 ? "on" : ""} /></div>
      <p className="xs mut">{step === 1 ? "Etapa 1 de 3 · Dados da identidade CRTPL" : step === 2 ? "Etapa 2 de 3 · Verificação do CDG" : "Etapa 3 de 3 · Senha do JOVEM+"}</p>

      {step === 1 && (
        <div key={shake} className={shake ? "shake" : ""}>
          <Field label="Nome completo" value={f.name} onChange={(v) => set("name", v)} placeholder="Como está na sua identidade CRTPL" error={errs.name} maxLength={40} />
          <Field label="Identificador pessoal CRTPL" value={f.identifier} onChange={(v) => set("identifier", maskIdInput(v))} placeholder="CRT-XXXX-XXXX" error={errs.identifier} />
          <Field label="E-mail JVPL" value={f.email} onChange={(v) => set("email", v.toLowerCase().replace(/\s/g, ""))} placeholder="nome@jvpl.demo" error={errs.email} type="email" />
          <Field label="Número demonstrativo (+55)" value={f.phone} onChange={(v) => set("phone", maskPhoneInput(v))} placeholder="(67) 90000-0000" inputMode="numeric" error={errs.phone} onEnter={step1} />
          <Msg />
          {L.rem > 0 && <div className="mt12"><LockBanner remaining={L.rem} /></div>}
          <div className="mt24"><Btn onClick={step1} loading={loading} disabled={L.rem > 0} icon={Search}>{loading ? "Consultando base CRTPL…" : "Continuar"}</Btn></div>
          <p className="xs mut mt12" style={{ textAlign: "center" }}>Tentativas restantes: <b>{L.rem > 0 ? 0 : left}</b> de {MAX_TRIES}</p>
          <div className="mt12"><SimBanner>Não tem identidade? Crie uma no app CRTPL. Os dados precisam ser da <b>mesma</b> identidade.</SimBanner></div>
        </div>
      )}

      {step === 2 && (
        <div key={shake} className={shake ? "shake" : ""}>
          <div className="card mt16 col" style={{ alignItems: "center", gap: 8 }}>
            <ShieldCheck size={30} color="var(--ac)" />
            <div className="b">Digite o CDG da sua identidade</div>
            <div className="xs mut" style={{ textAlign: "center" }}>Está na área de identidade do CRTPL.</div>
          </div>
          <Field label="CDG — Código de Garantia" className="cdgin" value={cdg} inputMode="numeric" maxLength={4} placeholder="••••" onChange={(v) => { setCdg(digitsOf(v).slice(0, 4)); setMsg(null); }} onEnter={step2} autoFocus disabled={L.rem > 0 || loading} />
          <Msg />
          {L.rem > 0 && <div className="mt12"><LockBanner remaining={L.rem} /></div>}
          <div className="mt24"><Btn onClick={step2} loading={loading && !(msg && msg.t === "ok")} disabled={L.rem > 0 || (msg && msg.t === "ok")} icon={Fingerprint}>{loading && !(msg && msg.t === "ok") ? "Verificando…" : "Verificar identidade"}</Btn></div>
          <p className="xs mut mt12" style={{ textAlign: "center" }}>Tentativas restantes: <b>{L.rem > 0 ? 0 : left}</b> de {MAX_TRIES}</p>
          {msg && msg.text.includes("Entrar") && <div className="mt12"><Btn kind="sec" onClick={() => go("login")}>Ir para Entrar</Btn></div>}
        </div>
      )}

      {step === 3 && (
        <>
          <div className="banner ok mt16"><CircleCheck size={18} /><div><b>Identidade verificada.</b> Crie uma senha própria para o JOVEM+, diferente da senha do CRTPL.</div></div>
          <Field label="Senha do JOVEM+" secret value={p1} onChange={(v) => { setP1(v); setMsg(null); }} placeholder="Mínimo de 6 caracteres" />
          <Field label="Confirmar senha" secret value={p2} onChange={(v) => { setP2(v); setMsg(null); }} onEnter={step3} />
          <button className="row g12 mt16" style={{ textAlign: "left" }} onClick={() => setAck(!ack)}>
            <span className={`tog ${ack ? "on" : ""}`} />
            <span className="sm">Entendo que o JOVEM+ é uma simulação e não movimenta dinheiro real.</span>
          </button>
          <Msg />
          <div className="mt24"><Btn onClick={step3} loading={loading} icon={Check}>Criar minha conta</Btn></div>
        </>
      )}
    </div>
  );
}

function JvLogin({ getDb, commit, go, setJvUser }) {
  const [key, setKey] = useState("");
  const [pass, setPass] = useState("");
  const [err, setErr] = useState("");
  const [loading, setLoading] = useState(false);
  const [shake, setShake] = useState(0);
  const L = useLock();
  const submit = async () => {
    setErr("");
    if (!key.trim() || !pass) return setErr("Informe seu nome, e-mail JVPL ou identificador e a senha do JOVEM+.");
    setLoading(true);
    await wait(900);
    const d = getDb();
    const ident = findIdentity(d, key);
    const acct = ident && accountOf(d, ident.id);
    if (!ident) { setLoading(false); setShake((s) => s + 1); return setErr("Conta não encontrada. Confira os dados ou crie sua conta."); }
    if (!acct) { setLoading(false); setShake((s) => s + 1); return setErr("Esta identidade ainda não tem conta JOVEM+. Toque em Criar conta."); }
    const bucket = "jvlogin:" + acct.id;
    const li = lockInfo(d, bucket, Date.now());
    if (li.locked) { setLoading(false); return L.lock(li.remaining); }
    const h = await hashPass(pass, acct.salt);
    if (h === acct.passHash) {
      commit((x) => ({ db: clearLock(x, bucket) }));
      setLoading(false);
      return setJvUser(acct.id);
    }
    const r = commit((x) => failResult(x, bucket, Date.now()));
    setLoading(false);
    setShake((s) => s + 1);
    if (r.status === "locked") L.lock(r.remaining);
    else setErr(`Senha incorreta. Tentativas restantes: ${r.left}.`);
  };
  return (
    <div className="scroll">
      <Back onClick={() => go("welcome")} />
      <div className="row g12"><JvLogo size={44} /><h1>Entrar</h1></div>
      <p className="mut mt8">Use sua senha do JOVEM+ (não a do CRTPL).</p>
      <div key={shake} className={shake ? "shake" : ""}>
        <Field label="Nome, e-mail JVPL ou identificador" value={key} onChange={(v) => { setKey(v); setErr(""); }} placeholder="Ex.: nome@jvpl.demo" />
        <Field label="Senha do JOVEM+" secret value={pass} onChange={(v) => { setPass(v); setErr(""); }} onEnter={submit} />
      </div>
      {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
      {L.rem > 0 && <div className="mt12"><LockBanner remaining={L.rem} /></div>}
      <div className="mt24"><Btn onClick={submit} loading={loading} disabled={L.rem > 0} icon={Lock}>Entrar</Btn></div>
      <div className="row sb mt16">
        <button className="link" onClick={() => go("forgot")}>Esqueci minha senha</button>
        <button className="link" onClick={() => go("register")}>Criar conta</button>
      </div>
    </div>
  );
}

function JvForgot({ getDb, commit, go, toast }) {
  const [step, setStep] = useState(0);
  const [key, setKey] = useState("");
  const [cdg, setCdg] = useState("");
  const [p1, setP1] = useState("");
  const [p2, setP2] = useState("");
  const [err, setErr] = useState("");
  const [loading, setLoading] = useState(false);
  const [target, setTarget] = useState(null);
  const L = useLock();
  useEffect(() => { const li = lockInfo(getDb(), "jovem-recover", Date.now()); if (li.locked) L.lock(li.remaining); }, []);
  const verify = async () => {
    setErr("");
    if (!key.trim() || cdg.length !== 4) return setErr("Informe o nome, e-mail ou identificador e o CDG de 4 dígitos.");
    setLoading(true);
    await wait(1000);
    const d = getDb();
    const ident = findIdentity(d, key);
    const acct = ident && accountOf(d, ident.id);
    const r = commit((x) => verifyCDG(x, acct ? ident : null, cdg, Date.now(), "jovem-recover"));
    setLoading(false);
    if (r.status === "ok") { setTarget({ acctId: acct.id, ident }); setStep(1); }
    else if (r.status === "locked") L.lock(r.remaining);
    else setErr(`${MSG_FAIL} Tentativas restantes: ${r.left}.`);
  };
  const save = async () => {
    if (p1.length < 6) return setErr("A senha precisa ter pelo menos 6 caracteres.");
    if (p1 !== p2) return setErr("A confirmação não é igual à senha.");
    setLoading(true);
    if ((await hashPass(p1, target.ident.salt)) === target.ident.passHash) { setLoading(false); return setErr("Use uma senha diferente da senha do CRTPL."); }
    const salt = newSalt();
    const hash = await hashPass(p1, salt);
    await wait(800);
    commit((d) => ({ db: patchAcct(d, target.acctId, (a) => ({ ...a, passHash: hash, salt })) }));
    setLoading(false);
    toast("Senha do JOVEM+ redefinida.", "ok");
    go("login");
  };
  return (
    <div className="scroll">
      <Back onClick={() => go("welcome")} />
      <h1>Recuperar senha</h1>
      <p className="mut mt8" style={{ lineHeight: 1.5 }}>Demonstrativo: consultamos a identidade CRTPL vinculada à conta. Nenhuma mensagem real é enviada.</p>
      <div className="stepper mt12"><i className="on" /><i className={step ? "on" : ""} /></div>
      {step === 0 ? (
        <>
          <Field label="Nome, e-mail JVPL ou identificador" value={key} onChange={(v) => { setKey(v); setErr(""); }} />
          <Field label="CDG da identidade CRTPL" value={cdg} inputMode="numeric" maxLength={4} placeholder="0000" onChange={(v) => { setCdg(digitsOf(v).slice(0, 4)); setErr(""); }} onEnter={verify} />
          {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
          {L.rem > 0 && <div className="mt12"><LockBanner remaining={L.rem} /></div>}
          <div className="mt24"><Btn onClick={verify} loading={loading} disabled={L.rem > 0} icon={ShieldCheck}>Consultar identidade</Btn></div>
        </>
      ) : (
        <>
          <div className="banner ok mt12"><CircleCheck size={18} /><div>Identidade de <b>{firstName(target.ident.name)}</b> confirmada. Defina a nova senha do JOVEM+.</div></div>
          <Field label="Nova senha" secret value={p1} onChange={(v) => { setP1(v); setErr(""); }} />
          <Field label="Confirmar nova senha" secret value={p2} onChange={(v) => { setP2(v); setErr(""); }} onEnter={save} />
          {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
          <div className="mt24"><Btn onClick={save} loading={loading} icon={Lock}>Salvar nova senha</Btn></div>
        </>
      )}
    </div>
  );
}
const EDU = [
  ["Como funciona o crédito", "Crédito é dinheiro que você usa agora e paga depois, dentro de um limite aprovado. No cartão, a compra entra na fatura e vence numa data combinada. Pagar o valor total evita juros; pagar só o mínimo transforma o restante em dívida cara.", "Trate o limite como um teto, não como renda extra."],
  ["Como funciona o débito", "No débito, o valor sai direto do seu saldo na hora da compra. Sem saldo, não há compra. Não gera juros nem fatura, o que ajuda a controlar os gastos.", "Confira o saldo antes de comprar e guarde os comprovantes."],
  ["Como organizar o dinheiro", "Uma referência conhecida: 50% para necessidades, 30% para desejos e 20% para guardar ou quitar dívidas. Ajuste à sua realidade. Anote entradas e saídas por 30 dias antes de decidir onde cortar.", "O que é pequeno e frequente costuma pesar mais que o que é grande e raro."],
  ["Como funciona o parcelamento", "Parcelar divide um valor em pagamentos mensais. Sem juros, o total é igual ao preço; com juros, você paga mais. Antes de parcelar, some todas as parcelas que já assumiu para ver quanto da sua renda já está comprometido.", "Use o simulador de parcelas na aba Crédito para ver o custo total."],
  ["Juros", "Juros são o preço de usar dinheiro no tempo. Nos juros compostos, a taxa incide sobre o valor mais os juros anteriores. Por isso dívidas crescem rápido (e investimentos também). Uma taxa de 2% ao mês equivale a cerca de 26,8% ao ano.", "Compare sempre o custo total, não só o valor da parcela."],
  ["Planejamento financeiro", "Defina metas com valor e prazo: R$ 600 em 6 meses são R$ 100 por mês. Comece pela reserva de emergência, equivalente a alguns meses de gastos essenciais. Se possível, separe o valor assim que receber, antes de gastar.", "Meta pequena e constante vence meta grande e esquecida."],
  ["Cuidados com golpes", "Desconfie de pressa, prêmios inesperados e pedidos de senha, token ou código. Banco de verdade nunca pede senha nem manda transferir dinheiro \"por segurança\". Confira o endereço do site, não clique em links de desconhecidos e use somente aplicativos oficiais.", "Na dúvida, pare, respire e ligue para o número oficial do seu banco."],
  ["Responsabilidade financeira", "Gastar com consciência é separar o que você precisa do que você quer. Evite emprestar seu nome ou cartão, e peça ajuda cedo se as parcelas apertarem: renegociar logo costuma custar menos.", "Dinheiro é ferramenta: planejar hoje dá liberdade amanhã."],
];
const COURSE_CATS = ["Tecnologia", "Programação", "Mecatrônica", "Informática", "Administração", "Empreendedorismo", "Cursos gratuitos", "Jovem aprendiz"];
const COURSES = [
  { id: "c1", t: "Lógica de programação do zero", cats: ["Tecnologia", "Programação", "Cursos gratuitos"], lv: "Iniciante", h: 20, d: "Algoritmos, fluxogramas e raciocínio para resolver problemas passo a passo." },
  { id: "c2", t: "Primeiros passos em Python", cats: ["Programação", "Tecnologia"], lv: "Iniciante", h: 30, d: "Variáveis, laços e funções com pequenos projetos práticos." },
  { id: "c3", t: "Eletrônica e Arduino na prática", cats: ["Mecatrônica", "Tecnologia"], lv: "Iniciante", h: 40, d: "Circuitos, sensores e atuadores para montar seus primeiros protótipos." },
  { id: "c4", t: "Automação e robótica básica", cats: ["Mecatrônica"], lv: "Intermediário", h: 60, d: "Controle de motores, leitura de sensores e noções de automação industrial." },
  { id: "c5", t: "Informática básica e pacote de escritório", cats: ["Informática", "Cursos gratuitos"], lv: "Iniciante", h: 24, d: "Sistema operacional, internet segura, texto, planilha e apresentação." },
  { id: "c6", t: "Manutenção de computadores", cats: ["Informática"], lv: "Intermediário", h: 48, d: "Montagem, diagnóstico de falhas e instalação de sistemas." },
  { id: "c7", t: "Rotinas administrativas", cats: ["Administração", "Jovem aprendiz"], lv: "Iniciante", h: 40, d: "Atendimento, arquivo, comunicação profissional e organização de tarefas." },
  { id: "c8", t: "Finanças pessoais para jovens", cats: ["Administração", "Cursos gratuitos"], lv: "Iniciante", h: 12, d: "Orçamento, metas, crédito consciente e como evitar dívidas." },
  { id: "c9", t: "Do zero ao primeiro negócio", cats: ["Empreendedorismo"], lv: "Iniciante", h: 30, d: "Identificar oportunidades, validar uma ideia e precificar." },
  { id: "c10", t: "Modelo de negócio em uma página", cats: ["Empreendedorismo", "Cursos gratuitos"], lv: "Iniciante", h: 16, d: "Monte e teste um plano simples com clientes reais ou imaginários." },
  { id: "c11", t: "Jovem aprendiz: como se preparar", cats: ["Jovem aprendiz", "Cursos gratuitos"], lv: "Iniciante", h: 8, d: "Entenda como funciona o programa, documentos comuns e postura profissional." },
  { id: "c12", t: "Currículo e entrevista para o primeiro emprego", cats: ["Jovem aprendiz", "Administração"], lv: "Iniciante", h: 6, d: "Como escrever um currículo claro e se preparar para conversas de seleção." },
];
const CAUSES = [
  { id: "k1", name: "Fundo de bolsas (simulado)" },
  { id: "k2", name: "Kits de robótica (simulado)" },
  { id: "k3", name: "Biblioteca itinerante (simulado)" },
];
const INCL = {
  Oportunidades: [["Voluntariado em projetos de bairro", "Ações locais de educação, esporte e cultura em busca de ajuda."], ["Mentoria para jovens", "Conversas com profissionais sobre carreira e estudos."], ["Inclusão digital", "Oficinas para aprender a usar computador e internet com segurança."], ["Feiras de oportunidades", "Eventos com vagas de estágio e aprendizagem na sua região."]],
  Bolsas: [["Bolsa de estudo (exemplo)", "Apoio para mensalidade de cursos técnicos. Modelo ilustrativo."], ["Auxílio material escolar", "Ajuda para livros e equipamentos. Modelo ilustrativo."], ["Bolsa de iniciação tecnológica", "Pesquisa e prática com orientação. Modelo ilustrativo."], ["Bolsa de idiomas", "Curso de inglês ou espanhol. Modelo ilustrativo."]],
  Projetos: [["Clube de robótica", "Equipes montam e programam robôs para desafios."], ["Oficina de programação comunitária", "Aulas abertas em centros comunitários e bibliotecas."], ["Horta educativa", "Aprender ciência e trabalho em equipe cultivando alimentos."], ["Biblioteca itinerante", "Livros e atividades de leitura levados a bairros distantes."]],
};

function TxRow({ tx, onOpen }) {
  const out = tx.type === "out";
  const Ic = tx.kind === "pix" ? Zap : tx.kind === "card" ? CreditCard : tx.kind === "donation" ? HeartHandshake : tx.kind === "fatura" ? Receipt : Banknote;
  return (
    <button className="irow" style={{ width: "100%", textAlign: "left" }} onClick={() => onOpen(tx)}>
      <div className="ico"><Ic size={19} color={out ? "var(--tx)" : "var(--ok)"} /></div>
      <div className="grow"><div className="b sm ell">{tx.title}</div><div className="xs mut ell">{tx.counter.name} · {fmtDay(tx.date)}</div></div>
      <b style={{ color: out ? "var(--tx)" : "var(--ok)", fontSize: 14 }}>{out ? "−" : "+"}{brl(tx.amount)}</b>
    </button>
  );
}

/* ================= INÍCIO ================= */
function JvHome({ X }) {
  const { acct, ident, go, openReceipt, setBell } = X;
  const [hide, setHide] = useState(false);
  const month = new Date().getMonth();
  const mTx = acct.txs.filter((t) => new Date(t.date).getMonth() === month);
  const inn = mTx.filter((t) => t.type === "in").reduce((s, t) => s + t.amount, 0);
  const outt = mTx.filter((t) => t.type === "out").reduce((s, t) => s + t.amount, 0);
  const used = usedCredit(acct);
  const unread = acct.notifs.filter((n) => !n.read).length;
  const V = (v) => (hide ? "R$ ••••" : v);
  return (
    <div className="scroll" style={{ padding: 0 }}>
      <div className="top">
        <div className="topbar">
          <JvLogo size={34} light />
          <div className="row g8">
            <button className="iconbtn" aria-label="Notificações" onClick={() => setBell(true)}><Bell size={19} />{unread > 0 && acct.notifOn && <i className="dot" />}</button>
            <button className="iconbtn" aria-label="Minha conta" onClick={() => go("more", "account")}><User size={19} /></button>
          </div>
        </div>
        <div className="mt24 rise">
          <h1 style={{ fontSize: 27 }}>Olá, {firstName(ident.name)}! 👋</h1>
          <p style={{ opacity: 0.85, marginTop: 4 }}>Que bom ter você por aqui.</p>
        </div>
      </div>
      <div className="float card rise" style={{ animationDelay: ".1s" }}>
        <div className="row sb"><span className="mut sm">Saldo demonstrativo</span><button className="eye" style={{ position: "static", width: 32, height: 32 }} aria-label={hide ? "Mostrar valores" : "Ocultar valores"} onClick={() => setHide(!hide)}>{hide ? <EyeOff size={18} /> : <Eye size={18} />}</button></div>
        <div className="b" style={{ fontSize: 34, letterSpacing: "-.03em", marginTop: -2 }}>{V(brl(acct.balance))}</div>
        <div className="sep" />
        <div className="row" style={{ gap: 8 }}>
          <div className="grow"><div className="xs mut">Limite disponível</div><div className="b sm">{V(brl(acct.creditLimit - used))}</div></div>
          <div className="grow"><div className="xs mut row g4"><TrendingUp size={12} color="var(--ok)" />Entradas do mês</div><div className="b sm">{V(brl(inn))}</div></div>
          <div className="grow"><div className="xs mut row g4"><TrendingDown size={12} color="var(--er)" />Saídas do mês</div><div className="b sm">{V(brl(outt))}</div></div>
        </div>
      </div>
      <div style={{ padding: "18px 18px 28px" }}>
        <div className="row" style={{ gap: 10 }}>
          {[["Pix", Zap, () => go("pix")], ["Cartões", CreditCard, () => go("cards")], ["Parcelas", Percent, () => go("credit")], ["Aprender", BookOpen, () => go("more", "edu")]].map(([l, I, fn]) => (
            <button key={l} className="tile grow" onClick={fn}><span className="ti"><I size={21} /></span>{l}</button>
          ))}
        </div>

        <div className="row sb mt24"><h3>Crédito</h3><button className="link" onClick={() => go("credit")}>Ver detalhes</button></div>
        <div className="card mt8">
          <div className="row sb sm"><span className="mut">Usado {V(brl(used))}</span><span className="b">Limite {V(brl(acct.creditLimit))}</span></div>
          <div className="bar mt8"><i style={{ width: `${Math.min(100, (used / acct.creditLimit) * 100)}%` }} /></div>
        </div>

        <div className="row sb mt24"><h3>Cartões</h3><button className="link" onClick={() => go("cards")}>Gerenciar</button></div>
        <div className="card mt8" style={{ padding: "4px 16px" }}>
          {[["Débito demonstrativo", "debit"], ["Crédito demonstrativo", "credit"], ["Cartão virtual demonstrativo", "virtual"]].map(([l, k]) => (
            <div key={k} className="irow"><div className="ico"><CreditCard size={19} color="var(--ac)" /></div><div className="grow b sm">{l}</div><span className={`badge ${acct.cards[k].blocked ? "er" : ""}`}>{acct.cards[k].blocked ? "Bloqueado" : "Ativo"}</span></div>
          ))}
        </div>

        <div className="row sb mt24"><h3>Últimas movimentações</h3><button className="link" onClick={() => go("more", "account")}>Ver tudo</button></div>
        <div className="card mt8" style={{ padding: "4px 16px" }}>
          {acct.txs.length === 0 ? <Empty icon={Receipt} title="Sem movimentações" text="Quando você usar o Pix ou os cartões, elas aparecem aqui." /> : acct.txs.slice(0, 5).map((t) => <TxRow key={t.id} tx={t} onOpen={openReceipt} />)}
        </div>

        <div className="row sb mt24"><h3>Notificações</h3><button className="link" onClick={() => setBell(true)}>Abrir</button></div>
        <div className="card mt8" style={{ padding: "4px 16px" }}>
          {acct.notifs.length === 0 ? <Empty icon={Bell} title="Tudo em dia" text="Você não tem notificações." /> : acct.notifs.slice(0, 2).map((n) => (
            <div key={n.id} className="irow"><div className="ico"><Bell size={18} color={n.read ? "var(--mut)" : "var(--ac)"} /></div><div className="grow"><div className="b sm">{n.title}</div><div className="xs mut">{n.text}</div></div></div>
          ))}
        </div>
        <div className="mt16"><SimBanner>Saldos, limites e movimentações são fictícios.</SimBanner></div>
      </div>
    </div>
  );
}

/* ================= PIX ================= */
function JvPix({ X }) {
  const [tab, setTab] = useState("send");
  return (
    <div className="scroll">
      <h1>Pix</h1>
      <div className="mt8"><SimBanner>Pix <b>demonstrativo</b>: não usa o sistema Pix real e não movimenta dinheiro.</SimBanner></div>
      <div className="seg mt16">
        {[["send", "Enviar"], ["recv", "Receber"], ["keys", "Chaves"], ["hist", "Histórico"]].map(([k, l]) => <button key={k} className={tab === k ? "on" : ""} onClick={() => setTab(k)}>{l}</button>)}
      </div>
      {tab === "send" && <PixSend X={X} />}
      {tab === "recv" && <PixReceive X={X} />}
      {tab === "keys" && <PixKeys X={X} />}
      {tab === "hist" && <PixHistory X={X} />}
    </div>
  );
}
function PixSend({ X }) {
  const { db, acct, ident, commit, toast, openReceipt } = X;
  const [key, setKey] = useState("");
  const [rec, setRec] = useState(null);
  const [cents, setCents] = useState(0);
  const [desc, setDesc] = useState("");
  const [err, setErr] = useState("");
  const [confirm, setConfirm] = useState(false);
  const [loading, setLoading] = useState(false);
  const dir = directory(db).filter((d) => d.accountId !== acct.id);
  const search = (k = key) => {
    const r = findRecipient(db, acct.id, k);
    if (!r) { setRec(null); setErr("Chave não encontrada na base demonstrativa (ou é a sua própria chave)."); return; }
    setErr(""); setRec(r);
  };
  const review = () => {
    if (!rec) return setErr("Escolha um destinatário.");
    if (!cents) return setErr("Informe um valor.");
    if (cents > acct.balance) return setErr(`Saldo insuficiente. Você tem ${brl(acct.balance)}.`);
    setErr(""); setConfirm(true);
  };
  const send = async () => {
    setLoading(true);
    await wait(1200);
    const r = commit((d) => sendPix(d, acct.id, rec, cents, desc.trim(), Date.now()));
    setLoading(false); setConfirm(false);
    if (r.error) return setErr(r.error);
    openReceipt(r.tx, r.balanceAfter);
    setRec(null); setKey(""); setCents(0); setDesc(""); setErr("");
    toast("Pix simulado enviado!", "ok");
  };
  return (
    <div>
      {!rec ? (
        <>
          <Field label="Chave do destinatário" value={key} onChange={(v) => { setKey(v); setErr(""); }} placeholder="E-mail JVPL, celular, identificador ou chave aleatória" onEnter={() => search()} />
          {err && <div className="err mt8"><AlertTriangle size={15} />{err}</div>}
          <div className="mt12"><Btn onClick={() => search()} icon={Search}>Buscar destinatário</Btn></div>
          <h3 className="mt24">Contatos fictícios</h3>
          <div className="card mt8" style={{ padding: "4px 16px" }}>
            {dir.map((d) => (
              <button key={d.identifier} className="irow" style={{ width: "100%", textAlign: "left" }} onClick={() => { setKey(d.keys[0]); search(d.keys[0]); }}>
                <div className="ico b">{d.name[0]}</div>
                <div className="grow"><div className="b sm">{d.name}</div><div className="xs mut">{maskIdent(d.identifier)} · {d.kind === "account" ? "Conta JOVEM+" : "Contato fictício"}</div></div>
                <ChevronRight size={18} className="mut" />
              </button>
            ))}
          </div>
        </>
      ) : (
        <>
          <div className="card mt16 row g12">
            <div className="ico b" style={{ width: 48, height: 48 }}>{rec.name[0]}</div>
            <div className="grow"><div className="b">{rec.name}</div><div className="xs mut">Identificador {maskIdent(rec.identifier)}</div></div>
            <button className="link" onClick={() => { setRec(null); setErr(""); }}>Trocar</button>
          </div>
          <MoneyInput cents={cents} onChange={(v) => { setCents(v); setErr(""); }} label={`Valor (saldo ${brl(acct.balance)})`} />
          <Field label="Descrição (opcional)" value={desc} onChange={setDesc} maxLength={60} placeholder="Ex.: lanche" />
          {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
          <div className="mt24"><Btn onClick={review} icon={Send}>Revisar Pix</Btn></div>
        </>
      )}
      {confirm && (
        <Confirm title="Confirmar Pix simulado?" confirmLabel="Confirmar" loading={loading} onConfirm={send} onCancel={() => setConfirm(false)}
          text={<div className="col g8"><div>Para <b>{rec.name}</b> ({maskIdent(rec.identifier)})</div><div className="b" style={{ fontSize: 24, color: "var(--tx)" }}>{brl(cents)}</div>{desc && <div>“{desc}”</div>}<div>Saldo restante: <b>{brl(acct.balance - cents)}</b></div><div className="xs">Simulação. Nenhum dinheiro real é enviado.</div></div>} />
      )}
    </div>
  );
}
function PixReceive({ X }) {
  const { db, acct, ident, commit, toast } = X;
  const keys = [ident.email, ident.phone, ident.identifier, ...acct.rndKeys];
  const [sel, setSel] = useState(0);
  const [cents, setCents] = useState(0);
  const [loading, setLoading] = useState(false);
  const key = keys[Math.min(sel, keys.length - 1)];
  const label = (k) => (k === ident.email ? "E-mail" : k === ident.phone ? "Celular" : k === ident.identifier ? "Identificador" : "Aleatória");
  const code = `SIMULACAO.PIX.${key}${cents ? "." + cents : ""}`;
  const simulate = async () => {
    if (!cents) return toast("Informe um valor para simular o recebimento.", "er");
    setLoading(true);
    await wait(1000);
    const from = SEEDS[Math.floor(Math.random() * SEEDS.length)];
    const r = commit((d) => receivePixSim(d, acct.id, cents, from, Date.now()));
    setLoading(false);
    if (r.error) return toast(r.error, "er");
    toast(`${from.name} pagou ${brl(cents)} (simulação).`, "ok");
    setCents(0);
  };
  return (
    <div>
      <div className="chips mt16">{keys.map((k, i) => <button key={k} className={`chip ${i === sel ? "on" : ""}`} onClick={() => setSel(i)}>{label(k)}</button>)}</div>
      <MoneyInput cents={cents} onChange={setCents} label="Valor da cobrança (opcional)" />
      <div className="card mt16 col" style={{ alignItems: "center", gap: 12 }}>
        <FakeQR seed={code} />
        <div className="xs mut" style={{ textAlign: "center", wordBreak: "break-all" }}>{code}</div>
        <CopyBtn text={code} label="Copiar código demonstrativo" toast={toast} />
      </div>
      <div className="mt16"><Btn kind="sec" icon={RefreshCw} loading={loading} onClick={simulate}>Simular recebimento</Btn></div>
      <p className="xs mut mt8" style={{ textAlign: "center" }}>O QR é decorativo e não funciona em nenhum app de banco.</p>
    </div>
  );
}
function PixKeys({ X }) {
  const { acct, ident, commit, toast } = X;
  const [del, setDel] = useState(null);
  const fixed = [["E-mail JVPL", ident.email, Mail], ["Celular demonstrativo", fmtPhone(ident.phone), Smartphone], ["Identificador CRTPL", ident.identifier, Fingerprint]];
  const add = () => { const r = commit((d) => addRandomKey(d, acct.id)); r.error ? toast(r.error, "er") : toast("Chave aleatória demonstrativa criada.", "ok"); };
  return (
    <div>
      <div className="card mt16" style={{ padding: "4px 16px" }}>
        {fixed.map(([l, v, I]) => (
          <div className="irow" key={l}><div className="ico"><I size={18} color="var(--ac)" /></div><div className="grow"><div className="xs mut">{l}</div><div className="b sm ell">{v}</div></div><CopyBtn text={v} label="" toast={toast} /></div>
        ))}
        {acct.rndKeys.map((k) => (
          <div className="irow" key={k}><div className="ico"><Key size={18} color="var(--ac)" /></div><div className="grow"><div className="xs mut">Chave aleatória</div><div className="b sm ell">{k}</div></div>
            <CopyBtn text={k} label="" toast={toast} /><button className="eye" style={{ position: "static" }} aria-label="Excluir chave" onClick={() => setDel(k)}><Trash2 size={18} color="var(--er)" /></button></div>
        ))}
      </div>
      <div className="mt16"><Btn kind="sec" icon={Plus} onClick={add}>Gerar chave aleatória</Btn></div>
      <p className="xs mut mt8">As chaves de e-mail, celular e identificador vêm da sua identidade CRTPL e não podem ser removidas. Todas são fictícias.</p>
      {del && <Confirm danger title="Excluir chave?" text="Essa chave aleatória demonstrativa deixará de receber Pix simulados." confirmLabel="Excluir" onCancel={() => setDel(null)} onConfirm={() => { commit((d) => ({ db: removeRandomKey(d, acct.id, del) })); setDel(null); toast("Chave excluída.", "in"); }} />}
    </div>
  );
}
function PixHistory({ X }) {
  const { acct, openReceipt } = X;
  const list = acct.txs.filter((t) => t.kind === "pix");
  return (
    <div className="card mt16" style={{ padding: "4px 16px" }}>
      {list.length === 0 ? <Empty icon={Zap} title="Nenhum Pix ainda" text="Envie ou receba um Pix demonstrativo para ver o histórico e os comprovantes." /> : list.map((t) => <TxRow key={t.id} tx={t} onOpen={openReceipt} />)}
    </div>
  );
}

/* ================= CARTÕES ================= */
function PCard({ type, acct, ident, showId }) {
  const blk = acct.cards[type] && acct.cards[type].blocked;
  const v = acct.cards.virtual;
  const title = { debit: "Débito", credit: "Crédito", virtual: "Virtual", phys: "Físico" }[type];
  return (
    <div className={`pcard ${type} ${blk ? "blk" : ""}`}>
      <div className="wm">DEMO</div>
      <div className="row sb"><div className="row g8"><JvLogo size={26} light /><b>JOVEM+</b></div><b className="sm">{title}</b></div>
      {type === "phys" ? (
        <div className="col g8" style={{ alignItems: "flex-start" }}><Lock size={26} /><b>Indisponível</b><span className="xs">Cartão físico não é emitido na demonstração.</span></div>
      ) : (
        <>
          <div className="row sb"><div className="chipic" />{blk && <span className="badge er" style={{ background: "#fff", color: "#D9363E" }}>Bloqueado</span>}</div>
          <div>
            <div style={{ fontFamily: "ui-monospace,Menlo,monospace", letterSpacing: ".08em", fontSize: 15 }}>
              {type === "virtual" ? (showId ? v.id : `VRT-••••-${v.id.slice(-4)}`) : `DEMO •••• •••• ${acct.number.slice(0, 4)}`}
            </div>
            <div className="row sb xs mt8" style={{ opacity: 0.9 }}><span className="ell" style={{ maxWidth: 190 }}>{ident.name.toUpperCase()}</span><span>{type === "virtual" ? `Val. ${v.exp} (ilustrativa)` : "Sem validade real"}</span></div>
          </div>
        </>
      )}
    </div>
  );
}
function JvCards({ X }) {
  const { acct, ident, commit, toast, go } = X;
  const [sel, setSel] = useState("debit");
  const [showId, setShowId] = useState(false);
  const [conf, setConf] = useState(false);
  const [spend, setSpend] = useState(false);
  const c = acct.cards[sel];
  const toggle = () => { commit((d) => ({ db: setCardBlocked(d, acct.id, sel, !c.blocked) })); setConf(false); toast(c.blocked ? "Cartão desbloqueado." : "Cartão bloqueado.", "ok"); };
  return (
    <div className="scroll">
      <h1>Cartões</h1>
      <div className="chips mt16">{[["debit", "Débito"], ["credit", "Crédito"], ["virtual", "Virtual"], ["phys", "Físico"]].map(([k, l]) => <button key={k} className={`chip ${sel === k ? "on" : ""}`} onClick={() => { setSel(k); setShowId(false); }}>{l}</button>)}</div>
      <div className="mt8" key={sel} style={{ animation: "rise .35s both" }}><PCard type={sel} acct={acct} ident={ident} showId={showId} /></div>
      <div className="mt12"><SimBanner><b>Cartão demonstrativo — não permite pagamentos reais.</b> Não há número utilizável nem CVC.</SimBanner></div>
      {sel === "phys" ? (
        <div className="card mt16"><div className="row g12"><Lock size={20} className="mut" /><div className="sm"><b>Cartão físico indisponível</b><div className="mut">A emissão de cartões físicos não existe nesta simulação.</div></div></div></div>
      ) : (
        <div className="col g12 mt16">
          <div className="card">
            <div className="irow"><div className="grow"><div className="xs mut">Status</div><span className={`badge ${c.blocked ? "er" : ""}`}>{c.blocked ? "Bloqueado" : "Desbloqueado"}</span></div>
              <button className={`tog ${c.blocked ? "" : "on"}`} aria-label={c.blocked ? "Desbloquear cartão" : "Bloquear cartão"} onClick={() => setConf(true)} /></div>
            {sel === "virtual" && (
              <>
                <div className="irow"><div className="grow"><div className="xs mut">Identificador fictício exclusivo</div><div className="b sm" style={{ fontFamily: "ui-monospace,Menlo,monospace" }}>{showId ? c.id : `VRT-••••-${c.id.slice(-4)}`}</div></div>
                  <button className="eye" style={{ position: "static" }} aria-label="Mostrar identificador" onClick={() => setShowId(!showId)}>{showId ? <EyeOff size={18} /> : <Eye size={18} />}</button></div>
                <div className="irow"><div className="grow"><div className="xs mut">Últimos 4 caracteres</div><div className="b sm">{c.id.slice(-4)}</div></div></div>
                <div className="irow"><div className="grow"><div className="xs mut">Titular · Validade ilustrativa</div><div className="b sm">{ident.name} · {c.exp}</div></div></div>
              </>
            )}
          </div>
          {sel === "virtual" && <CopyBtn text={c.id} label="Copiar identificador fictício" toast={toast} />}
          {sel === "debit" && <Btn kind="sec" icon={Banknote} onClick={() => setSpend(true)}>Simular compra no débito</Btn>}
          {sel === "credit" && <Btn kind="sec" icon={Percent} onClick={() => go("credit")}>Ver limite e parcelas</Btn>}
        </div>
      )}
      {conf && <Confirm title={c.blocked ? "Desbloquear cartão?" : "Bloquear cartão?"} text={c.blocked ? "O cartão voltará a aceitar compras simuladas." : "O cartão recusará compras simuladas até ser desbloqueado."} confirmLabel={c.blocked ? "Desbloquear" : "Bloquear"} danger={!c.blocked} onConfirm={toggle} onCancel={() => setConf(false)} />}
      {spend && <DebitSheet X={X} onClose={() => setSpend(false)} />}
    </div>
  );
}
function DebitSheet({ X, onClose }) {
  const { acct, commit, toast, openReceipt } = X;
  const [shop, setShop] = useState("Mercadinho do bairro");
  const [cents, setCents] = useState(0);
  const [err, setErr] = useState("");
  const [loading, setLoading] = useState(false);
  const go = async () => {
    setLoading(true);
    await wait(900);
    const r = commit((d) => debitSpend(d, acct.id, shop, cents, Date.now()));
    setLoading(false);
    if (r.error) return setErr(r.error);
    onClose(); openReceipt(r.tx, acct.balance - cents); toast("Compra simulada no débito.", "ok");
  };
  return (
    <Sheet title="Compra no débito (simulação)" onClose={onClose}>
      <div className="chips mt8">{["Mercadinho do bairro", "Lanchonete", "Papelaria", "Transporte"].map((s) => <button key={s} className={`chip ${shop === s ? "on" : ""}`} onClick={() => setShop(s)}>{s}</button>)}</div>
      <MoneyInput cents={cents} onChange={(v) => { setCents(v); setErr(""); }} />
      {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
      <div className="mt24"><Btn loading={loading} onClick={go} icon={Check} disabled={!cents}>Confirmar compra</Btn></div>
    </Sheet>
  );
}

/* ================= CRÉDITO ================= */
function JvCredit({ X }) {
  const { acct, commit, toast, openReceipt } = X;
  const used = usedCredit(acct);
  const avail = acct.creditLimit - used;
  const [simV, setSimV] = useState(0);
  const [simN, setSimN] = useState(6);
  const [purchase, setPurchase] = useState(null);
  const [payConf, setPayConf] = useState(false);
  const [antic, setAntic] = useState(null);
  const [loading, setLoading] = useState(false);
  const calc = simV ? calcInstallment(simV, simN) : null;
  const due = acct.purchases.filter((p) => p.paid < p.n).reduce((s, p) => s + p.monthly, 0);
  const pay = async () => {
    setLoading(true); await wait(1000);
    const r = commit((d) => payInvoice(d, acct.id, Date.now()));
    setLoading(false); setPayConf(false);
    if (r.error) return toast(r.error, "er");
    openReceipt(r.tx, acct.balance - r.due); toast("Fatura simulada paga.", "ok");
  };
  const doAnt = async () => {
    setLoading(true); await wait(1000);
    const r = commit((d) => anticipate(d, acct.id, antic.id, Date.now()));
    setLoading(false); setAntic(null);
    if (r.error) return toast(r.error, "er");
    toast(`Antecipação simulada: economia de ${brl(r.q.discount)}.`, "ok");
  };
  const q = antic ? anticipateQuote(antic) : null;
  return (
    <div className="scroll">
      <h1>Crédito</h1>
      <div className="card mt16">
        <div className="row sb"><span className="mut sm">Limite demonstrativo</span><b>{brl(acct.creditLimit)}</b></div>
        <div className="bar mt12"><i style={{ width: `${Math.min(100, (used / acct.creditLimit) * 100)}%` }} /></div>
        <div className="row sb mt12"><div><div className="xs mut">Utilizado</div><b>{brl(used)}</b></div><div style={{ textAlign: "right" }}><div className="xs mut">Disponível</div><b style={{ color: "var(--ok)" }}>{brl(avail)}</b></div></div>
        <div className="row g8 mt16">
          <Btn small kind="sec" icon={Plus} onClick={() => setPurchase({ v: 0, n: 1 })}>Compra fictícia</Btn>
          <Btn small kind="sec" icon={Receipt} onClick={() => (due ? setPayConf(true) : toast("Não há parcelas em aberto.", "in"))}>Pagar fatura</Btn>
        </div>
      </div>

      <h3 className="mt24">Simulador de parcelas</h3>
      <div className="card mt8">
        <MoneyInput cents={simV} onChange={setSimV} label="Valor da compra" />
        <div className="chips mt12">{[1, 2, 3, 6, 10, 12].map((n) => <button key={n} className={`chip ${simN === n ? "on" : ""}`} onClick={() => setSimN(n)}>{n}x</button>)}</div>
        {calc ? (
          <div className="mt12 col g8">
            <div className="row sb"><span className="mut sm">Parcela</span><b>{simN}x de {brl(calc.monthly)}</b></div>
            <div className="row sb"><span className="mut sm">Total a pagar</span><b>{brl(calc.total)}</b></div>
            <div className="row sb"><span className="mut sm">Juros</span><b>{calc.rate ? `${(calc.rate * 100).toFixed(2).replace(".", ",")}% ao mês · ${brl(calc.total - simV)}` : "Sem juros"}</b></div>
            <Btn small icon={Check} onClick={() => setPurchase({ v: simV, n: simN })}>Registrar como compra fictícia</Btn>
          </div>
        ) : <p className="xs mut mt12">Digite um valor para comparar parcelas. Até {FREE_INSTALLMENTS}x sem juros; acima disso, taxa ilustrativa de 1,99% ao mês.</p>}
      </div>

      <h3 className="mt24">Compras fictícias</h3>
      <div className="card mt8" style={{ padding: "4px 16px" }}>
        {acct.purchases.length === 0 ? <Empty icon={CreditCard} title="Nenhuma compra no crédito" text="Registre uma compra fictícia para ver as parcelas e testar a antecipação." /> : acct.purchases.map((p) => (
          <div key={p.id} className="irow" style={{ alignItems: "stretch", flexDirection: "column", gap: 8 }}>
            <div className="row sb"><div className="b sm">{p.desc}</div><b className="sm">{p.n}x {brl(p.monthly)}</b></div>
            <div className="bar" style={{ height: 7 }}><i style={{ width: `${(p.paid / p.n) * 100}%` }} /></div>
            <div className="row sb xs mut"><span>{p.paid}/{p.n} parcelas pagas · {fmtDay(p.date)}</span>
              {p.paid < p.n ? <button className="link" style={{ fontSize: 12.5 }} onClick={() => setAntic(p)}>Antecipar</button> : <span className="badge">Quitada</span>}</div>
          </div>
        ))}
      </div>
      <div className="mt16"><SimBanner>Todas as operações de crédito são simuladas e não movimentam dinheiro real.</SimBanner></div>

      {purchase && <PurchaseSheet X={X} init={purchase} onClose={() => setPurchase(null)} />}
      {payConf && <Confirm title="Pagar fatura simulada?" text={<>Será debitado <b>{brl(due)}</b> do saldo demonstrativo (uma parcela de cada compra).</>} confirmLabel="Pagar" onConfirm={pay} onCancel={() => setPayConf(false)} loading={loading} />}
      {antic && <Confirm title="Antecipar parcelas?" loading={loading} confirmLabel="Antecipar" onConfirm={doAnt} onCancel={() => setAntic(null)}
        text={<div className="col g8"><div>{antic.desc}: {q.rem} parcela(s) restantes ({brl(q.value)})</div><div>Desconto ilustrativo: <b>{brl(q.discount)}</b></div><div>Você paga agora: <b style={{ color: "var(--tx)" }}>{brl(q.pay)}</b></div></div>} />}
    </div>
  );
}
function PurchaseSheet({ X, init, onClose }) {
  const { acct, commit, toast } = X;
  const [desc, setDesc] = useState("Compra fictícia");
  const [v, setV] = useState(init.v);
  const [n, setN] = useState(init.n);
  const [err, setErr] = useState("");
  const [loading, setLoading] = useState(false);
  const c = v ? calcInstallment(v, n) : null;
  const save = async () => {
    setLoading(true); await wait(900);
    const r = commit((d) => addPurchase(d, acct.id, cleanName(desc) || "Compra fictícia", v, n, Date.now()));
    setLoading(false);
    if (r.error) return setErr(r.error);
    toast("Compra fictícia registrada.", "ok"); onClose();
  };
  return (
    <Sheet title="Compra fictícia no crédito" onClose={onClose}>
      <Field label="Descrição" value={desc} onChange={setDesc} maxLength={40} />
      <MoneyInput cents={v} onChange={(x) => { setV(x); setErr(""); }} label="Valor" />
      <div className="chips mt12">{[1, 2, 3, 6, 10, 12].map((x) => <button key={x} className={`chip ${n === x ? "on" : ""}`} onClick={() => { setN(x); setErr(""); }}>{x}x</button>)}</div>
      {c && <p className="sm mt8">{n}x de <b>{brl(c.monthly)}</b> · total {brl(c.total)}</p>}
      {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
      <div className="mt24"><Btn onClick={save} loading={loading} disabled={!v} icon={Check}>Registrar compra</Btn></div>
    </Sheet>
  );
}

/* ================= MAIS / CONTA / EDUCAÇÃO / CURSOS / INCLUSÃO ================= */
function JvMore({ X }) {
  const { go, exit, setConfirmOut } = X;
  const items = [["Minha conta", "Dados, histórico e configurações", User, () => go("more", "account")], ["Educação financeira", "Crédito, juros, golpes e mais", BookOpen, () => go("more", "edu")], ["Cursos e oportunidades", "Tecnologia, programação, jovem aprendiz", GraduationCap, () => go("more", "courses")], ["Inclusão e apoio", "Bolsas, projetos e doações simuladas", HeartHandshake, () => go("more", "incl")]];
  return (
    <div className="scroll">
      <h1>Mais</h1>
      <div className="card mt16" style={{ padding: "4px 16px" }}>
        {items.map(([t, s, I, fn]) => (
          <button key={t} className="irow" style={{ width: "100%", textAlign: "left" }} onClick={fn}>
            <div className="ico"><I size={20} color="var(--ac)" /></div><div className="grow"><div className="b">{t}</div><div className="xs mut">{s}</div></div><ChevronRight size={18} className="mut" />
          </button>
        ))}
      </div>
      <div className="col g12 mt24">
        <Btn kind="out" icon={Link2} onClick={X.goCrtpl}>Abrir o CRTPL</Btn>
        <Btn kind="danger" icon={LogOut} onClick={() => setConfirmOut(true)}>Sair do JOVEM+</Btn>
      </div>
    </div>
  );
}
function JvAccount({ X }) {
  const { acct, ident, commit, toast, openReceipt, setConfirmOut, go } = X;
  const [f, setF] = useState("all");
  const [pw, setPw] = useState(false);
  const list = acct.txs.filter((t) => f === "all" || t.type === f);
  return (
    <div className="scroll">
      <Back onClick={() => go("more")} />
      <div className="card row g12">
        <div className="ico b" style={{ width: 54, height: 54, borderRadius: 18, background: "var(--ac)", color: "#fff", fontSize: 22 }}>{firstName(ident.name)[0]}</div>
        <div className="grow"><h2 className="ell">{ident.name}</h2><div className="xs mut">Cliente desde {fmtDay(acct.createdAt)}</div></div>
      </div>
      <h3 className="mt24">Identidade CRTPL vinculada</h3>
      <div className="card mt8" style={{ padding: "4px 16px" }}>
        {[["Nome", ident.name], ["Identificador", maskIdent(ident.identifier)], ["E-mail JVPL", ident.email], ["Telefone", fmtPhone(ident.phone)]].map(([k, v]) => <div className="irow" key={k}><span className="mut sm" style={{ width: 98 }}>{k}</span><b className="sm ell grow">{v}</b></div>)}
        <div className="irow"><span className="mut sm" style={{ width: 98 }}>Status</span><span className="badge"><Link2 size={12} />Vinculada · {ident.status}</span></div>
      </div>
      <h3 className="mt24">Dados da conta fictícia</h3>
      <div className="card mt8" style={{ padding: "4px 16px" }}>
        {[["Banco", "JOVEM+ (demo)"], ["Agência", acct.agency], ["Conta", acct.number], ["Saldo", brl(acct.balance)]].map(([k, v]) => <div className="irow" key={k}><span className="mut sm" style={{ width: 98 }}>{k}</span><b className="sm grow">{v}</b></div>)}
      </div>
      <h3 className="mt24">Histórico</h3>
      <div className="seg mt8">{[["all", "Tudo"], ["in", "Entradas"], ["out", "Saídas"]].map(([k, l]) => <button key={k} className={f === k ? "on" : ""} onClick={() => setF(k)}>{l}</button>)}</div>
      <div className="card mt8" style={{ padding: "4px 16px" }}>{list.length === 0 ? <Empty icon={Receipt} title="Nada por aqui" text="Nenhuma movimentação neste filtro." /> : list.map((t) => <TxRow key={t.id} tx={t} onOpen={openReceipt} />)}</div>
      <h3 className="mt24">Configurações</h3>
      <div className="card mt8" style={{ padding: "4px 16px" }}>
        <div className="irow"><Bell size={18} className="mut" /><div className="grow b sm">Notificações</div>
          <button className={`tog ${acct.notifOn ? "on" : ""}`} aria-label="Alternar notificações" onClick={() => commit((d) => ({ db: patchAcct(d, acct.id, (a) => ({ ...a, notifOn: !a.notifOn })) }))} /></div>
        <button className="irow" style={{ width: "100%", textAlign: "left" }} onClick={() => setPw(true)}><Lock size={18} className="mut" /><div className="grow b sm">Alterar senha do JOVEM+</div><ChevronRight size={18} className="mut" /></button>
        <button className="irow" style={{ width: "100%", textAlign: "left", color: "var(--er)" }} onClick={() => setConfirmOut(true)}><LogOut size={18} /><div className="grow b sm">Sair</div></button>
      </div>
      {pw && <JvChangePass X={X} ident={ident} onClose={() => setPw(false)} />}
    </div>
  );
}
function JvChangePass({ X, ident, onClose }) {
  const { acct, commit, toast } = X;
  const [cur, setCur] = useState(""); const [p1, setP1] = useState(""); const [p2, setP2] = useState("");
  const [err, setErr] = useState(""); const [loading, setLoading] = useState(false);
  const save = async () => {
    setErr("");
    if (p1.length < 6) return setErr("A nova senha precisa ter pelo menos 6 caracteres.");
    if (p1 !== p2) return setErr("A confirmação não é igual à nova senha.");
    setLoading(true);
    const ok = (await hashPass(cur, acct.salt)) === acct.passHash;
    await wait(700);
    if (!ok) { setLoading(false); return setErr("Senha atual incorreta."); }
    if ((await hashPass(p1, ident.salt)) === ident.passHash) { setLoading(false); return setErr("Use uma senha diferente da senha do CRTPL."); }
    const salt = newSalt(); const hash = await hashPass(p1, salt);
    commit((d) => ({ db: patchAcct(d, acct.id, (a) => ({ ...a, passHash: hash, salt })) }));
    setLoading(false); toast("Senha do JOVEM+ alterada.", "ok"); onClose();
  };
  return (
    <Sheet title="Alterar senha do JOVEM+" onClose={onClose}>
      <Field label="Senha atual" secret value={cur} onChange={(v) => { setCur(v); setErr(""); }} />
      <Field label="Nova senha" secret value={p1} onChange={(v) => { setP1(v); setErr(""); }} />
      <Field label="Confirmar nova senha" secret value={p2} onChange={(v) => { setP2(v); setErr(""); }} onEnter={save} />
      {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
      <div className="mt24"><Btn onClick={save} loading={loading} icon={Lock}>Salvar</Btn></div>
    </Sheet>
  );
}
function JvEdu({ X }) {
  const [open, setOpen] = useState(0);
  return (
    <div className="scroll">
      <Back onClick={() => X.go("more")} />
      <h1>Educação financeira</h1>
      <p className="mut mt8">Conteúdos curtos para tomar boas decisões com o dinheiro.</p>
      {EDU.map(([t, body, tip], i) => (
        <div className="acc" key={t}>
          <button onClick={() => setOpen(open === i ? -1 : i)} aria-expanded={open === i}><div className="ico"><BookOpen size={18} color="var(--ac)" /></div><span className="grow">{t}</span><ChevronDown size={18} className="mut" style={{ transform: open === i ? "rotate(180deg)" : "", transition: ".2s" }} /></button>
          {open === i && <div className="body"><p>{body}</p><div className="banner mt12"><Sparkles size={16} /><div style={{ color: "var(--tx)" }}><b>Dica:</b> {tip}</div></div></div>}
        </div>
      ))}
    </div>
  );
}
function JvCourses({ X }) {
  const { acct, commit, toast } = X;
  const [cat, setCat] = useState("Todos");
  const list = COURSES.filter((c) => cat === "Todos" || c.cats.includes(cat));
  const toggle = (id) => { const on = acct.saved.includes(id); commit((d) => ({ db: patchAcct(d, acct.id, (a) => ({ ...a, saved: on ? a.saved.filter((x) => x !== id) : [...a.saved, id] })) })); toast(on ? "Removido dos seus interesses." : "Salvo nos seus interesses.", "ok"); };
  return (
    <div className="scroll">
      <Back onClick={() => X.go("more")} />
      <h1>Cursos e oportunidades</h1>
      <div className="chips mt16">{["Todos", ...COURSE_CATS].map((c) => <button key={c} className={`chip ${cat === c ? "on" : ""}`} onClick={() => setCat(c)}>{c}</button>)}</div>
      <p className="xs mut">{acct.saved.length} interesse(s) salvo(s)</p>
      {list.length === 0 ? <Empty icon={GraduationCap} title="Sem cursos nesta categoria" text="Escolha outra categoria." /> : list.map((c) => (
        <div key={c.id} className="card mt12">
          <div className="row sb g8"><h3>{c.t}</h3>{c.cats.includes("Cursos gratuitos") && <span className="badge">Gratuito</span>}</div>
          <p className="mut sm mt8" style={{ lineHeight: 1.45 }}>{c.d}</p>
          <div className="row sb mt12"><span className="xs mut">{c.lv} · {c.h}h · {c.cats[0]}</span>
            <Btn small kind={acct.saved.includes(c.id) ? "" : "sec"} icon={acct.saved.includes(c.id) ? Check : Bookmark} onClick={() => toggle(c.id)}>{acct.saved.includes(c.id) ? "Salvo" : "Tenho interesse"}</Btn></div>
        </div>
      ))}
      <div className="mt16"><SimBanner>Exemplos ilustrativos de temas de estudo. Não representam ofertas reais e não há vínculo, patrocínio ou parceria com empresas ou instituições.</SimBanner></div>
    </div>
  );
}
function JvIncl({ X }) {
  const { acct, commit, toast, openReceipt } = X;
  const [tab, setTab] = useState("Oportunidades");
  const [cause, setCause] = useState(null);
  const [cents, setCents] = useState(0);
  const [err, setErr] = useState("");
  const [loading, setLoading] = useState(false);
  const total = acct.donations.reduce((s, d) => s + d.amount, 0);
  const give = async () => {
    setLoading(true); await wait(1000);
    const r = commit((d) => donate(d, acct.id, cause, cents, Date.now()));
    setLoading(false);
    if (r.error) return setErr(r.error);
    setCause(null); setCents(0); openReceipt(r.tx, acct.balance - cents); toast("Doação simulada registrada.", "ok");
  };
  return (
    <div className="scroll">
      <Back onClick={() => X.go("more")} />
      <h1>Inclusão e apoio</h1>
      <div className="chips mt16">{["Oportunidades", "Bolsas", "Projetos", "Doações", "Transparência"].map((t) => <button key={t} className={`chip ${tab === t ? "on" : ""}`} onClick={() => setTab(t)}>{t}</button>)}</div>
      {INCL[tab] && INCL[tab].map(([t, d]) => (
        <div key={t} className="card mt12"><div className="row g12"><div className="ico"><HeartHandshake size={19} color="var(--ac)" /></div><div><b>{t}</b><p className="mut sm mt8" style={{ lineHeight: 1.45 }}>{d}</p></div></div></div>
      ))}
      {tab === "Doações" && (
        <>
          <div className="mt12"><SimBanner><b>Doações simuladas.</b> Apenas o saldo fictício é usado. Nenhuma organização recebe dinheiro e nenhuma chave Pix verdadeira é solicitada.</SimBanner></div>
          {CAUSES.map((c) => (
            <div key={c.id} className="card mt12 row g12"><div className="ico"><Gift size={19} color="var(--ac)" /></div><div className="grow b sm">{c.name}</div><Btn small icon={HeartHandshake} onClick={() => { setCause(c); setErr(""); }}>Doar</Btn></div>
          ))}
        </>
      )}
      {tab === "Transparência" && (
        <>
          <div className="card mt12"><div className="xs mut">Total doado por você (simulado)</div><div className="b" style={{ fontSize: 28 }}>{brl(total)}</div></div>
          <div className="card mt12">
            {CAUSES.map((c) => { const s = acct.donations.filter((d) => d.causeId === c.id).reduce((x, d) => x + d.amount, 0); return (
              <div key={c.id} className="mt8"><div className="row sb sm"><span>{c.name}</span><b>{brl(s)}</b></div><div className="bar mt8" style={{ height: 7 }}><i style={{ width: total ? `${(s / total) * 100}%` : 0 }} /></div></div>
            ); })}
          </div>
          <h3 className="mt16">Registro de doações</h3>
          <div className="card mt8" style={{ padding: "4px 16px" }}>
            {acct.donations.length === 0 ? <Empty icon={HeartHandshake} title="Sem doações" text="Faça uma doação simulada na aba Doações e acompanhe aqui." /> : acct.donations.map((d) => <div key={d.id} className="irow"><div className="grow"><div className="b sm">{d.cause}</div><div className="xs mut">{fmtDate(d.date)}</div></div><b>{brl(d.amount)}</b></div>)}
          </div>
          <div className="mt12"><SimBanner>Os números vêm só das suas simulações neste dispositivo. Servem para exercitar a ideia de prestação de contas.</SimBanner></div>
        </>
      )}
      {cause && (
        <Sheet title="Doação simulada" onClose={() => setCause(null)}>
          <p className="mut mt8">{cause.name}</p>
          <div className="chips mt12">{[500, 1000, 2000].map((v) => <button key={v} className="chip" onClick={() => { setCents(v); setErr(""); }}>{brl(v)}</button>)}</div>
          <MoneyInput cents={cents} onChange={(v) => { setCents(v); setErr(""); }} />
          {err && <div className="err mt12"><AlertTriangle size={15} />{err}</div>}
          <div className="mt24"><Btn loading={loading} disabled={!cents} icon={HeartHandshake} onClick={give}>Confirmar doação simulada</Btn></div>
        </Sheet>
      )}
    </div>
  );
}

/* ================= SHELL LOGADO ================= */
function JvMain({ db, commit, getDb, toast, acct, ident, onLogout, goCrtpl }) {
  const [tab, setTab] = useState("home");
  const [sub, setSub] = useState(null);
  const [receipt, setReceipt] = useState(null);
  const [bell, setBell] = useState(false);
  const [confirmOut, setConfirmOut] = useState(false);
  const go = (t, s = null) => { setTab(t); setSub(s); };
  const X = { db, commit, getDb, toast, acct, ident, go, goCrtpl, setBell, setConfirmOut, openReceipt: (tx, balanceAfter) => setReceipt({ tx, balanceAfter }) };
  const closeBell = () => { setBell(false); commit((d) => ({ db: patchAcct(d, acct.id, (a) => ({ ...a, notifs: a.notifs.map((n) => ({ ...n, read: true })) })) })); };
  let body;
  if (tab === "home") body = <JvHome X={X} />;
  else if (tab === "pix") body = <JvPix X={X} />;
  else if (tab === "cards") body = <JvCards X={X} />;
  else if (tab === "credit") body = <JvCredit X={X} />;
  else body = sub === "account" ? <JvAccount X={X} /> : sub === "edu" ? <JvEdu X={X} /> : sub === "courses" ? <JvCourses X={X} /> : sub === "incl" ? <JvIncl X={X} /> : <JvMore X={X} />;
  const tabs = [["home", "Início", Home], ["pix", "Pix", Zap], ["cards", "Cartões", CreditCard], ["credit", "Crédito", Banknote], ["more", "Mais", MoreHorizontal]];
  return (
    <>
      {body}
      <nav className="nav" aria-label="Navegação principal">
        {tabs.map(([k, l, I]) => <button key={k} className={tab === k ? "on" : ""} onClick={() => go(k)} aria-current={tab === k}><I size={21} />{l}</button>)}
      </nav>
      {receipt && <Receipt_ tx={receipt.tx} balanceAfter={receipt.balanceAfter} who={ident.name} onClose={() => setReceipt(null)} />}
      {bell && (
        <Sheet title="Notificações" onClose={closeBell}>
          {acct.notifs.length === 0 ? <Empty icon={Bell} title="Tudo em dia" text="Você não tem notificações." /> : acct.notifs.map((n) => (
            <div key={n.id} className="irow"><div className="ico"><Bell size={18} color={n.read ? "var(--mut)" : "var(--ac)"} /></div><div className="grow"><div className="b sm">{n.title}</div><div className="xs mut">{n.text}</div><div className="xs mut">{fmtDate(n.date)}</div></div></div>
          ))}
        </Sheet>
      )}
      {confirmOut && <Confirm title="Sair do JOVEM+?" text="Você precisará entrar novamente com a senha do JOVEM+." confirmLabel="Sair" onConfirm={onLogout} onCancel={() => setConfirmOut(false)} />}
    </>
  );
}

function JvApp({ db, commit, getDb, toast, jvUser, setJvUser, exit, goCrtpl }) {
  const [view, setView] = useState("welcome");
  const acct = jvUser ? db.accounts.find((a) => a.id === jvUser) : null;
  const ident = acct ? identOf(db, acct) : null;
  useEffect(() => { if (jvUser && (!acct || !ident)) setJvUser(null); }, [jvUser, acct, ident]);
  if (acct && ident) return <JvMain db={db} commit={commit} getDb={getDb} toast={toast} acct={acct} ident={ident} goCrtpl={goCrtpl} onLogout={() => { setJvUser(null); setView("welcome"); toast("Você saiu do JOVEM+.", "in"); }} />;
  if (view === "register") return <JvRegister db={db} getDb={getDb} commit={commit} go={setView} toast={toast} setJvUser={setJvUser} />;
  if (view === "login") return <JvLogin getDb={getDb} commit={commit} go={setView} setJvUser={setJvUser} />;
  if (view === "forgot") return <JvForgot getDb={getDb} commit={commit} go={setView} toast={toast} />;
  return <JvWelcome go={setView} exit={exit} />;
}
const CSS2 = `
.sel{--bg:#0E0B1A;--s1:#1A1530;--s2:#261F42;--tx:#F4F0FF;--mut:#A99FC9;--ac:#FFC24B;--onac:#2A1B00;--ln:#ffffff1c;--ok:#5FE3A1;--er:#FF7A88;--sh:0 10px 30px #0006}
.panel{position:relative;text-align:left;border-radius:26px;padding:20px;overflow:hidden;width:100%;transition:transform .15s;display:flex;flex-direction:column;gap:10px}
.panel:active{transform:scale(.985)}
.panel.p1{background:linear-gradient(150deg,#2A2147,#1C1630 70%);box-shadow:inset 0 0 0 1.5px #FFC24B66;color:#F4F0FF}
.panel.p2{background:linear-gradient(150deg,#0B2447,#1F6FEB 120%);box-shadow:inset 0 0 0 1.5px #19C37D88;color:#fff}
.link-pill{align-self:center;display:flex;align-items:center;gap:8px;padding:7px 14px;border-radius:99px;background:var(--s2);font-size:12.5px;font-weight:700;color:var(--ac);position:relative;margin:-14px 0;z-index:2;box-shadow:0 0 0 4px var(--bg)}
`;

async function persist(dbv) {
  try {
    if (typeof window === "undefined" || !window.storage) return false;
    const r = await window.storage.set(DB_KEY, JSON.stringify(dbv), false);
    return !!r;
  } catch (e) {
    return false;
  }
}

function Selector({ db, mode, onSelect, onTests, onReset }) {
  return (
    <div className="scroll">
      <div className="rise">
        <h1 style={{ fontSize: 30 }}>Duas experiências,<br />uma identidade.</h1>
        <p className="mut mt8" style={{ lineHeight: 1.5 }}>Escolha um aplicativo. Os dois compartilham a mesma base demonstrativa.</p>
      </div>
      <div className="col mt24" style={{ gap: 0 }}>
        <button className="panel p1 rise" style={{ animationDelay: ".08s" }} onClick={() => onSelect("crtpl")}>
          <div className="row sb"><CrLogo size={58} /><ChevronRight size={22} color="#FFC24B" /></div>
          <div><div className="b" style={{ fontSize: 28, letterSpacing: "-.03em" }}>CRTPL</div><div style={{ color: "#FFC24B", fontWeight: 700 }}>Carteira Pessoal</div></div>
          <p style={{ opacity: 0.8, fontSize: 14, lineHeight: 1.45 }}>Crie e administre sua identidade fictícia, com e-mail JVPL, telefone e CDG.</p>
          <div className="xs" style={{ opacity: 0.7 }}>{db.identities.length} identidade(s) na base</div>
        </button>
        <div className="link-pill"><Link2 size={15} />CDG conecta os dois</div>
        <button className="panel p2 rise" style={{ animationDelay: ".16s" }} onClick={() => onSelect("jovem")}>
          <div className="row sb"><JvLogo size={58} /><ChevronRight size={22} color="#19D98B" /></div>
          <div><div className="b" style={{ fontSize: 28, letterSpacing: "-.03em" }}>JOVEM<span style={{ color: "#19D98B" }}>+</span></div><div style={{ opacity: 0.92, fontWeight: 600 }}>Seu futuro começa agora.</div></div>
          <p style={{ opacity: 0.85, fontSize: 14, lineHeight: 1.45 }}>Banco demonstrativo com Pix, cartões, crédito, cursos e educação financeira.</p>
          <div className="xs" style={{ opacity: 0.8 }}>{db.accounts.length} conta(s) na base</div>
        </button>
      </div>
      <div className="mt24">
        {mode === "persistent" ? (
          <div className="banner ok"><ShieldCheck size={17} /><div><b>Armazenamento persistente ativo.</b> Identidades e contas continuam depois de atualizar a página. Logins não são mantidos: você entra de novo.</div></div>
        ) : (
          <div className="banner er"><AlertTriangle size={17} /><div><b>Sem armazenamento persistente neste ambiente.</b> Os dados valem só enquanto esta página estiver aberta. Ao atualizar, tudo é perdido.</div></div>
        )}
      </div>
      <div className="row g8 mt16">
        <Btn kind="sec" small icon={FlaskConical} onClick={onTests}>Testes automáticos</Btn>
        <Btn kind="danger" small icon={Trash2} onClick={onReset}>Limpar base</Btn>
      </div>
      <p className="xs mut mt16" style={{ lineHeight: 1.5 }}>Projeto de diversão e aprendizado. Tudo é fictício: nenhum documento, conta, e-mail, SMS, cartão ou Pix reais.</p>
    </div>
  );
}

function TestsScreen({ onBack }) {
  const [res, setRes] = useState(null);
  const run = () => { setRes(null); setTimeout(() => setRes(runTests()), 400); };
  useEffect(run, []);
  const ok = res && res.every((r) => r.ok);
  return (
    <div className="scroll">
      <Back onClick={onBack} />
      <h1>Testes automáticos</h1>
      <p className="mut mt8" style={{ lineHeight: 1.5 }}>Rodam a lógica real (criação, duplicidade, CDG, bloqueio, contas, Pix e crédito) numa base temporária. Sua base não é alterada.</p>
      {!res ? <div className="row g8 mt24 mut"><Loader2 className="spin" size={18} />Executando…</div> : (
        <>
          <div className={`banner ${ok ? "ok" : "er"} mt16`}>{ok ? <CircleCheck size={18} /> : <CircleX size={18} />}<div><b>{res.filter((r) => r.ok).length} de {res.length} testes passaram.</b></div></div>
          <div className="card mt12" style={{ padding: "4px 16px" }}>
            {res.map((r) => (
              <div key={r.name} className="irow" style={{ alignItems: "flex-start" }}>
                {r.ok ? <CircleCheck size={19} color="var(--ok)" style={{ flex: "none" }} /> : <CircleX size={19} color="var(--er)" style={{ flex: "none" }} />}
                <div className="grow sm"><div className="b">{r.name}</div>{r.detail && <div className="xs err" style={{ marginTop: 4 }}>{r.detail}</div>}</div>
              </div>
            ))}
          </div>
        </>
      )}
      <div className="mt16"><Btn kind="sec" icon={RefreshCw} onClick={run}>Executar de novo</Btn></div>
    </div>
  );
}

export default function App() {
  const dbRef = useRef(emptyDB());
  const [db, setDb] = useState(dbRef.current);
  const [mode, setMode] = useState("loading");
  const [screen, setScreen] = useState("select");
  const [crUser, setCrUser] = useState(null);
  const [jvUser, setJvUser] = useState(null);
  const [toastS, setToastS] = useState(null);
  const [resetConf, setResetConf] = useState(false);
  const tRef = useRef(null);
  const queue = useRef(Promise.resolve());

  const persistNow = useCallback(() => {
    queue.current = queue.current.then(async () => {
      const ok = await persist(dbRef.current);
      setMode(ok ? "persistent" : "memory");
    });
  }, []);

  useEffect(() => {
    (async () => {
      let loaded = null;
      try {
        if (typeof window !== "undefined" && window.storage) {
          const r = await window.storage.get(DB_KEY, false);
          if (r && r.value) loaded = JSON.parse(r.value);
        }
      } catch (e) { /* chave inexistente ou armazenamento indisponível */ }
      if (loaded && Array.isArray(loaded.identities) && Array.isArray(loaded.accounts)) {
        loaded.locks = loaded.locks || {};
        dbRef.current = loaded;
        setDb(loaded);
      }
      await wait(1000);
      persistNow();
    })();
  }, [persistNow]);

  const commit = useCallback((fn) => {
    const r = fn(dbRef.current) || {};
    if (r.db && r.db !== dbRef.current) {
      dbRef.current = r.db;
      setDb(r.db);
      persistNow();
    }
    return r;
  }, [persistNow]);
  const getDb = useCallback(() => dbRef.current, []);
  const toast = useCallback((text, type = "in") => {
    clearTimeout(tRef.current);
    setToastS({ text, type, k: Date.now() });
    tRef.current = setTimeout(() => setToastS(null), 2600);
  }, []);

  const resetAll = () => {
    dbRef.current = emptyDB();
    setDb(dbRef.current);
    setCrUser(null);
    setJvUser(null);
    persistNow();
    setResetConf(false);
    toast("Base demonstrativa limpa.", "ok");
  };

  const theme = screen === "crtpl" ? "cr" : screen === "jovem" ? "jv" : "sel";
  const common = { db, commit, getDb, toast };
  return (
    <div className="shell">
      <style>{CSS + CSS2}</style>
      <div className={`phone ${theme}`}>
        {mode === "loading" ? (
          <div className="col ctr grow" style={{ alignItems: "center", justifyContent: "center", gap: 18, background: "#0E0B1A", color: "#F4F0FF" }}>
            <div className="row g16"><div className="logo-wrap"><CrLogo size={64} /></div><div className="logo-wrap" style={{ animationDelay: ".5s" }}><JvLogo size={64} /></div></div>
            <div className="row g8" style={{ color: "#A99FC9" }}><Loader2 className="spin" size={18} />Carregando base demonstrativa…</div>
          </div>
        ) : screen === "select" ? (
          <Selector db={db} mode={mode} onSelect={setScreen} onTests={() => setScreen("tests")} onReset={() => setResetConf(true)} />
        ) : screen === "tests" ? (
          <TestsScreen onBack={() => setScreen("select")} />
        ) : screen === "crtpl" ? (
          <CrApp {...common} crUser={crUser} setCrUser={setCrUser} exit={() => setScreen("select")} goJovem={() => setScreen("jovem")} />
        ) : (
          <JvApp {...common} jvUser={jvUser} setJvUser={setJvUser} exit={() => setScreen("select")} goCrtpl={() => setScreen("crtpl")} />
        )}
        {toastS && (
          <div key={toastS.k} className={`toast ${toastS.type}`} role="status">
            {toastS.type === "ok" ? <CircleCheck size={19} /> : toastS.type === "er" ? <CircleX size={19} /> : <Info size={19} />}
            <span>{toastS.text}</span>
          </div>
        )}
        {resetConf && <Confirm danger title="Limpar toda a base?" text="Apaga todas as identidades CRTPL e contas JOVEM+ criadas por você. Não dá para desfazer." confirmLabel="Limpar tudo" onConfirm={resetAll} onCancel={() => setResetConf(false)} />}
      </div>
    </div>
  );
}
