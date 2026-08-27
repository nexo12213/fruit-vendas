const path = require("path");
const express = require("express");
const cookieParser = require("cookie-parser");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcryptjs");
const { Pool } = require("pg");

const app = express();
const PORT = Number(process.env.PORT || 3000);
const ADMIN_USER = process.env.ADMIN_USER || "admin";
const ADMIN_PASSWORD = process.env.ADMIN_PASSWORD || "troque-esta-senha";
const JWT_SECRET = process.env.JWT_SECRET || "troque-esta-chave-secreta";
const PIX_KEY = process.env.PIX_KEY || "02326770692";
const DATABASE_URL = process.env.DATABASE_URL;

if (!DATABASE_URL) {
  console.error("DATABASE_URL não configurada.");
  process.exit(1);
}

const pool = new Pool({
  connectionString: DATABASE_URL,
  ssl: process.env.NODE_ENV === "production" ? { rejectUnauthorized: false } : false
});

app.use(express.json({ limit: "1mb" }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, "public")));

function cents(v) {
  return Math.round(Number(v) * 100);
}

function auth(req, res, next) {
  try {
    const token = req.cookies.nexo_admin;
    if (!token) return res.status(401).json({ error: "Não autenticado" });
    req.admin = jwt.verify(token, JWT_SECRET);
    next();
  } catch {
    return res.status(401).json({ error: "Sessão expirada" });
  }
}

function rowToOrder(r) {
  return {
    id: r.id,
    email: r.email,
    user: r.roblox_user || "",
    items: r.items_json,
    total: Number(r.total_cents) / 100,
    status: r.status,
    date: new Date(r.created_at).toLocaleString("pt-BR"),
    createdAt: r.created_at,
    updatedAt: r.updated_at
  };
}

async function initDb() {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS orders (
      id TEXT PRIMARY KEY,
      email TEXT NOT NULL,
      roblox_user TEXT,
      items_json JSONB NOT NULL,
      total_cents INTEGER NOT NULL,
      status TEXT NOT NULL CHECK (status IN ('Pendente','PIX confirmado','Entregue')),
      created_at TIMESTAMPTZ NOT NULL,
      updated_at TIMESTAMPTZ NOT NULL
    );
    CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(status);
    CREATE INDEX IF NOT EXISTS idx_orders_created_at ON orders(created_at);
  `);
}

app.get("/health", (req, res) => res.json({ ok: true }));

app.get("/api/config", (req, res) => {
  res.json({ pixKey: PIX_KEY });
});

app.post("/api/admin/login", async (req, res) => {
  const { user, password } = req.body || {};
  if (user !== ADMIN_USER) {
    return res.status(401).json({ error: "Usuário ou senha incorretos" });
  }

  const hash = bcrypt.hashSync(ADMIN_PASSWORD, 10);
  const ok = await bcrypt.compare(String(password || ""), hash);

  if (!ok) {
    return res.status(401).json({ error: "Usuário ou senha incorretos" });
  }

  const token = jwt.sign(
    { sub: ADMIN_USER, role: "admin" },
    JWT_SECRET,
    { expiresIn: "8h" }
  );

  res.cookie("nexo_admin", token, {
    httpOnly: true,
    sameSite: "lax",
    secure: process.env.NODE_ENV === "production",
    maxAge: 8 * 60 * 60 * 1000
  });

  res.json({ ok: true });
});

app.post("/api/admin/logout", (req, res) => {
  res.clearCookie("nexo_admin");
  res.json({ ok: true });
});

app.get("/api/admin/me", auth, (req, res) => {
  res.json({ user: req.admin.sub });
});

app.post("/api/orders", async (req, res) => {
  try {
    const { email, user, items, total } = req.body || {};

    if (
      !email ||
      !Array.isArray(items) ||
      !items.length ||
      !Number.isFinite(Number(total))
    ) {
      return res.status(400).json({ error: "Dados do pedido inválidos" });
    }

    const now = new Date();
    const id = "NX" + Date.now().toString().slice(-8);

    await pool.query(
      `INSERT INTO orders
       (id,email,roblox_user,items_json,total_cents,status,created_at,updated_at)
       VALUES ($1,$2,$3,$4,$5,$6,$7,$8)`,
      [
        id,
        String(email).trim(),
        String(user || "").trim(),
        JSON.stringify(items),
        cents(total),
        "Pendente",
        now,
        now
      ]
    );

    res.status(201).json({ id, status: "Pendente", pixKey: PIX_KEY });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: "Não foi possível criar o pedido." });
  }
});

app.get("/api/orders", auth, async (req, res) => {
  try {
    const { rows } = await pool.query(
      "SELECT * FROM orders ORDER BY created_at DESC"
    );
    res.json(rows.map(rowToOrder));
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: "Não foi possível carregar os pedidos." });
  }
});

app.patch("/api/orders/:id/status", auth, async (req, res) => {
  try {
    const next = req.body?.status;
    const allowed = ["Pendente", "PIX confirmado", "Entregue"];

    if (!allowed.includes(next)) {
      return res.status(400).json({ error: "Status inválido" });
    }

    const now = new Date();
    const result = await pool.query(
      "UPDATE orders SET status=$1, updated_at=$2 WHERE id=$3 RETURNING *",
      [next, now, req.params.id]
    );

    if (!result.rowCount) {
      return res.status(404).json({ error: "Pedido não encontrado" });
    }

    res.json(rowToOrder(result.rows[0]));
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: "Não foi possível alterar o status." });
  }
});

app.delete("/api/orders/:id", auth, async (req, res) => {
  try {
    const result = await pool.query(
      "DELETE FROM orders WHERE id=$1",
      [req.params.id]
    );

    if (!result.rowCount) {
      return res.status(404).json({ error: "Pedido não encontrado" });
    }

    res.json({ ok: true });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: "Não foi possível excluir o pedido." });
  }
});

app.use((req, res) => {
  res.sendFile(path.join(__dirname, "public", "index.html"));
});

(async () => {
  try {
    await initDb();
    app.listen(PORT, "0.0.0.0", () => {
      console.log(`NEXO Vendas rodando na porta ${PORT}`);
    });
  } catch (e) {
    console.error("Falha ao iniciar banco:", e);
    process.exit(1);
  }
})();
