mkdir -p src/middleware src/services tests .github/workflows

cat << 'EOF' > package.json
{
  "name": "express-api-jest-ci",
  "version": "1.0.0",
  "description": "Express API with Jest, Supertest and GitHub Actions CI",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "test": "jest",
    "test:coverage": "jest --coverage"
  },
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^7.0.0"
  }
}
EOF

cat << 'EOF' > jest.config.js
module.exports = {
  testEnvironment: 'node',
  coverageDirectory: 'coverage',
  collectCoverageFrom: [
    'app.js',
    'src/**/*.js'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
EOF

cat << 'EOF' > src/middleware/auth.js
const authMiddleware = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ message: 'Token taqdim etilmagan (Unauthorized)' });
  }

  const token = authHeader.split(' ')[1];
  if (token === 'user1-token') {
    req.user = { id: 'user1', role: 'user' };
    return next();
  }
  if (token === 'user2-token') {
    req.user = { id: 'user2', role: 'user' };
    return next();
  }

  return res.status(401).json({ message: 'Noto‘g‘ri token' });
};

module.exports = authMiddleware;
EOF

cat << 'EOF' > src/services/orderService.js
const orders = [
  { id: '1', userId: 'user1', item: 'Noutbuk', price: 1000, quantity: 1, total: 1000 },
  { id: '2', userId: 'user2', item: 'Klaviatura', price: 100, quantity: 2, total: 200 }
];

const calculateTotal = (price, quantity, discount = 0) => {
  if (price < 0 || quantity <= 0) {
    throw new Error('Narx yoki miqdor noto‘g‘ri');
  }
  const subtotal = price * quantity;
  return subtotal - (subtotal * discount) / 100;
};

const getAllOrders = async () => {
  return orders;
};

const getOrderById = async (id) => {
  return orders.find((o) => o.id === id) || null;
};

const createOrder = async ({ userId, item, price, quantity, discount = 0 }) => {
  const total = calculateTotal(price, quantity, discount);
  const newOrder = {
    id: String(orders.length + 1),
    userId,
    item,
    price,
    quantity,
    total
  };
  orders.push(newOrder);
  return newOrder;
};

module.exports = {
  calculateTotal,
  getAllOrders,
  getOrderById,
  createOrder
};
EOF

cat << 'EOF' > app.js
const express = require('express');
const authMiddleware = require('./src/middleware/auth');
const orderService = require('./src/services/orderService');

const app = express();
app.use(express.json());

// Ochiq route: Health check
app.get('/api/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Himoyalangan route: Barcha buyurtmalarni olish
app.get('/api/orders', authMiddleware, async (req, res) => {
  try {
    const orders = await orderService.getAllOrders();
    res.status(200).json(orders);
  } catch (error) {
    res.status(500).json({ message: 'Serverda ichki xatolik yuz berdi' });
  }
});

// Himoyalangan route: ID bo'yicha buyurtmani olish (404 va 403 tekshiruvi bilan)
app.get('/api/orders/:id', authMiddleware, async (req, res) => {
  try {
    const order = await orderService.getOrderById(req.params.id);
    if (!order) {
      return res.status(404).json({ message: 'Buyurtma topilmadi' });
    }
    if (order.userId !== req.user.id) {
      return res.status(403).json({ message: 'Boshqa foydalanuvchi buyurtmasiga ruxsat yo‘q' });
    }
    res.status(200).json(order);
  } catch (error) {
    res.status(500).json({ message: 'Serverda ichki xatolik yuz berdi' });
  }
});

// Himoyalangan route: Yangi buyurtma yaratish (400 validatsiya bilan)
app.post('/api/orders', authMiddleware, async (req, res) => {
  try {
    const { item, price, quantity, discount } = req.body;
    if (!item || typeof price !== 'number' || price <= 0 || typeof quantity !== 'number' || quantity <= 0) {
      return res.status(400).json({ message: 'Noto‘g‘ri ma‘lumotlar kiritildi' });
    }
    const created = await orderService.createOrder({
      userId: req.user.id,
      item,
      price,
      quantity,
      discount
    });
    res.status(201).json(created);
  } catch (error) {
    res.status(500).json({ message: 'Serverda ichki xatolik yuz berdi' });
  }
});

module.exports = app;
EOF

cat << 'EOF' > server.js
const app = require('./app');

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server ${PORT}-portda ishga tushdi`);
});
EOF

cat << 'EOF' > tests/orders.test.js
const request = require('supertest');
const app = require('../app');
const orderService = require('../src/services/orderService');

// Unit test: test.each bilan calculateTotal funksiyasini tekshirish (5-talab)
describe('calculateTotal unit testlari', () => {
  test.each([
    [100, 2, 0, 200],
    [250, 4, 10, 900],
    [500, 1, 50, 250],
    [120, 3, 0, 360]
  ])('narx=%i, miqdor=%i, chegirma=%i%% => jami=%i', (price, quantity, discount, expected) => {
    expect(orderService.calculateTotal(price, quantity, discount)).toBe(expected);
  });

  test('manfiy narx yoki 0 miqdor kiritilganda xato qaytarishi kerak', () => {
    expect(() => orderService.calculateTotal(-10, 2)).toThrow();
    expect(() => orderService.calculateTotal(100, 0)).toThrow();
  });
});

// Integration / Route testlari (2, 3, 4-talablar)
describe('Express API Route testlari', () => {
  afterEach(() => {
    jest.restoreAllMocks();
  });

  // 1-route testi: Muvaffaqiyatli health check (200)
  test('GET /api/health - 200 OK qaytarishi kerak', async () => {
    const res = await request(app).get('/api/health');
    expect(res.statusCode).toBe(200);
    expect(res.body).toEqual({ status: 'ok' });
  });

  // 2-route testi: Tokensiz so'rov (401)
  test('GET /api/orders - tokensiz so‘rov uchun 401 qaytarishi kerak', async () => {
    const res = await request(app).get('/api/orders');
    expect(res.statusCode).toBe(401);
    expect(res.body).toHaveProperty('message');
  });

  // 3-route testi: Noto'g'ri token bilan so'rov (401)
  test('GET /api/orders - yaroqsiz token uchun 401 qaytarishi kerak', async () => {
    const res = await request(app)
      .get('/api/orders')
      .set('Authorization', 'Bearer invalid-token');
    expect(res.statusCode).toBe(401);
  });

  // 4-route testi: Muvaffaqiyatli ro'yxatni olish (200)
  test('GET /api/orders - yaroqli token bilan buyurtmalar ro‘yxatini qaytarishi kerak (200)', async () => {
    const res = await request(app)
      .get('/api/orders')
      .set('Authorization', 'Bearer user1-token');
    expect(res.statusCode).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);
    expect(res.body.length).toBeGreaterThan(0);
  });

  // 5-route testi: O'zining buyurtmasini ID bo'yicha olish (200)
  test('GET /api/orders/1 - foydalanuvchi o‘z buyurtmasini olishi kerak (200)', async () => {
    const res = await request(app)
      .get('/api/orders/1')
      .set('Authorization', 'Bearer user1-token');
    expect(res.statusCode).toBe(200);
    expect(res.body.id).toBe('1');
  });

  // 6-route testi: Boshqa foydalanuvchi yozuviga kirish (403)
  test('GET /api/orders/2 - boshqa foydalanuvchi buyurtmasi uchun 403 qaytarishi kerak', async () => {
    const res = await request(app)
      .get('/api/orders/2')
      .set('Authorization', 'Bearer user1-token');
    expect(res.statusCode).toBe(403);
    expect(res.body).toHaveProperty('message');
  });

  // 7-route testi: Mavjud bo'lmagan ID (404)
  test('GET /api/orders/999 - mavjud bo‘lmagan buyurtma uchun 404 qaytarishi kerak', async () => {
    const res = await request(app)
      .get('/api/orders/999')
      .set('Authorization', 'Bearer user1-token');
    expect(res.statusCode).toBe(404);
    expect(res.body.message).toBe('Buyurtma topilmadi');
  });

  // 8-route testi: Validatsiya xatosi (400)
  test('POST /api/orders - noto‘g‘ri ma‘lumotlar bilan 400 qaytarishi kerak', async () => {
    const res = await request(app)
      .post('/api/orders')
      .set('Authorization', 'Bearer user1-token')
      .send({ item: '', price: -50, quantity: 0 });
    expect(res.statusCode).toBe(400);
    expect(res.body).toHaveProperty('message');
  });

  // 9-route testi: Yangi buyurtma yaratish (201)
  test('POST /api/orders - to‘g‘ri ma‘lumotlar bilan yangi buyurtma yaratishi kerak (201)', async () => {
    const res = await request(app)
      .post('/api/orders')
      .set('Authorization', 'Bearer user1-token')
      .send({ item: 'Sichqoncha', price: 50, quantity: 2, discount: 10 });
    expect(res.statusCode).toBe(201);
    expect(res.body.total).toBe(90);
  });

  // 10-route testi: jest.spyOn / mock orqali 500 server xatosi (4-talab)
  test('GET /api/orders - ma‘lumotlar qatlami xato berganda 500 qaytarishi kerak', async () => {
    jest.spyOn(orderService, 'getAllOrders').mockRejectedValue(new Error('DB connection lost'));

    const res = await request(app)
      .get('/api/orders')
      .set('Authorization', 'Bearer user1-token');

    expect(res.statusCode).toBe(500);
    expect(res.body.message).toBe('Serverda ichki xatolik yuz berdi');
  });
});
EOF

cat << 'EOF' > .github/workflows/test.yml
name: Express API CI Tests

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run Jest tests with coverage
        run: npm run test:coverage
EOF

cat << 'EOF' > .gitignore
node_modules/
coverage/
.env
EOF

cat << 'EOF' > README.md
# Express API — Jest, Supertest va GitHub Actions CI

Ushbu loyiha Express API uchun avtomatlashtirilgan unit va integratsion testlar to'plamini hamda GitHub Actions CI pipeline'ini o'z ichiga oladi.

## Loyiha tuzilmasi
- `app.js` — Express ilovasi va route'lar (testlash uchun eksport qilingan)
- `server.js` — Serverni ishga tushirish (`app.listen`)
- `src/middleware/auth.js` — Bearer token autentifikatsiya middleware'i
- `src/services/orderService.js` — Biznes mantiq va ma'lumotlar qatlami
- `tests/orders.test.js` — Jest + Supertest testlari (200, 201, 400, 401, 403, 404, 500 va `test.each`)
- `jest.config.js` — Jest va `coverageThreshold` (80%+) sozlamalari
- `.github/workflows/test.yml` — GitHub Actions CI konfiguratsiyasi

## Ishga tushirish va testlash
```bash
npm ci
npm run test:coverage
git remote add origin https://github.com/SIZNING_USERNAME/REPO_NOMI.git
git push -u origin main --force
