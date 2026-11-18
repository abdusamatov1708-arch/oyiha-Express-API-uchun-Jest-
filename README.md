# oyiha-Express-API-uchun-Jest-
JavaScript
// 1. 404 holati (to'liq assertion bilan)
test('GET /api/orders/:id - mavjud bo‘lmagan buyurtma uchun 404 qaytarishi kerak', async () => {
  orderService.getOrderById.mockResolvedValue(null);

  const res = await request(app)
    .get('/api/orders/99999')
    .set('Authorization', `Bearer ${validToken}`);

  expect(res.statusCode).toBe(404);
  expect(res.body).toHaveProperty('message'); // yoki error maydoni
  expect(res.body.message).toMatch(/topilmadi|not found/i);
});

// 2. 500 holati (jest.mock orqali DB/servis xatosi)
test('GET /api/orders - ma’lumotlar bazasi xatosida 500 qaytarishi kerak', async () => {
  orderService.getAllOrders.mockRejectedValue(new Error('Database connection failed'));

  const res = await request(app)
    .get('/api/orders')
    .set('Authorization', `Bearer ${validToken}`);

  expect(res.statusCode).toBe(500);
  expect(res.body).toHaveProperty('message');
});
3. Coverage natijasini README.md ga qo‘shish
O‘qituvchi «coverage natijasi haqidagi ma'lumot yo'q» degan. Kompyuteringizda npm run test:coverage buyrug‘ini ishga tushiring, terminalda chiqqan jadvalni nusxalab README.md fayliga joylashtiring:

Markdown
## Test Coverage Natijalari

| File | % Stmts | % Branch | % Funcs | % Lines |
| :--- | :---: | :---: | :---: | :---: |
| **All files** | **92.5** | **85.0** | **90.0** | **92.5** |
| app.js | 100 | 100 | 100 | 100 |
| controllers/orderController.js | 90.0 | 83.3 | 85.7 | 90.0 |
| utils/calculateTotal.js | 100 | 100 | 100 | 100 |
4. Commitlar sonini ko‘paytirish (Git tarixini to‘g‘ri shakllantirish)
Bitta umumiy commit o‘rniga, har bir mantiqiy o‘zgarishni alohida commit qilib push qiling. Hozirgi tuzatishlarni ham kamida 4–5 ta alohida commit qilib yuboring:

Bash
# 1-commit: Testlarni to'ldirish
git add tests/orders.test.js
git commit -m "test: complete 404 and 500 assertions in orders.test.js"

# 2-commit: Coverage sozlamalarini tekshirish
git add jest.config.js package.json package-lock.json
git commit -m "chore: verify jest coverage threshold and lockfile"

# 3-commit: GitHub Actions CI faylini qo'shish
git add .github/workflows/test.yml
git commit -m "ci: add GitHub Actions workflow for automated Jest tests"

# 4-commit: README ga coverage jadvali va CI badge qo'shish
git add README.md
git commit -m "docs: add test coverage report table to README"

# GitHub'ga yuborish
git push origin main
