# Advanced E2E Testing Patterns

This document covers sophisticated testing scenarios and patterns.

## Scenario 1: Authentication & Authorization Testing

### Login Flow Testing

```typescript
describe('Authentication E2E', () => {
  let app: INestApplication;
  let userRepository: UserRepository;

  beforeAll(async () => {
    // Setup...
  });

  describe('POST /auth/register', () => {
    it('should register new user', async () => {
      const res = await request(app.getHttpServer())
        .post('/auth/register')
        .send({
          email: 'newuser@test.com',
          password: 'SecurePassword123!',
          name: 'New User',
        })
        .expect(201);

      expect(res.body).toHaveProperty('id');
      expect(res.body.email).toBe('newuser@test.com');

      // Verify saved in database
      const user = await userRepository.findByEmail('newuser@test.com');
      expect(user).toBeDefined();
      expect(user.passwordHash).toBeDefined();
      expect(user.passwordHash).not.toBe('SecurePassword123!'); // Should be hashed
    });
  });

  describe('POST /auth/login', () => {
    beforeEach(async () => {
      // Create test user
      await userRepository.save({
        email: 'user@test.com',
        passwordHash: await hashPassword('Password123!'),
        name: 'Test User',
      });
    });

    it('should login with valid credentials', async () => {
      const res = await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'user@test.com',
          password: 'Password123!',
        })
        .expect(200);

      expect(res.body).toHaveProperty('accessToken');
      expect(res.body).toHaveProperty('refreshToken');
      expect(res.body.user.email).toBe('user@test.com');
    });

    it('should reject invalid password', async () => {
      await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'user@test.com',
          password: 'WrongPassword',
        })
        .expect(401);
    });

    it('should reject non-existent user', async () => {
      await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'nonexistent@test.com',
          password: 'Password123!',
        })
        .expect(401);
    });
  });

  describe('Protected Endpoints', () => {
    let authToken: string;
    let userId: string;

    beforeEach(async () => {
      // Create and login user
      const registerRes = await request(app.getHttpServer())
        .post('/auth/register')
        .send({
          email: 'user@test.com',
          password: 'Password123!',
          name: 'Test User',
        });

      userId = registerRes.body.id;

      const loginRes = await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'user@test.com',
          password: 'Password123!',
        });

      authToken = loginRes.body.accessToken;
    });

    it('should access protected endpoint with token', async () => {
      await request(app.getHttpServer())
        .get('/users/me')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200)
        .expect(res => {
          expect(res.body.id).toBe(userId);
          expect(res.body.email).toBe('user@test.com');
        });
    });

    it('should reject without token', async () => {
      await request(app.getHttpServer())
        .get('/users/me')
        .expect(401);
    });

    it('should reject with invalid token', async () => {
      await request(app.getHttpServer())
        .get('/users/me')
        .set('Authorization', 'Bearer invalid_token')
        .expect(401);
    });

    it('should reject with expired token', async () => {
      const expiredToken = generateExpiredToken();

      await request(app.getHttpServer())
        .get('/users/me')
        .set('Authorization', `Bearer ${expiredToken}`)
        .expect(401);
    });
  });

  describe('Role-Based Access Control', () => {
    let adminToken: string;
    let userToken: string;

    beforeEach(async () => {
      // Create admin
      await userRepository.save({
        email: 'admin@test.com',
        passwordHash: await hashPassword('Admin123!'),
        role: UserRole.ADMIN,
      });

      // Create regular user
      await userRepository.save({
        email: 'user@test.com',
        passwordHash: await hashPassword('User123!'),
        role: UserRole.USER,
      });

      // Get tokens
      const adminLogin = await request(app.getHttpServer())
        .post('/auth/login')
        .send({ email: 'admin@test.com', password: 'Admin123!' });

      adminToken = adminLogin.body.accessToken;

      const userLogin = await request(app.getHttpServer())
        .post('/auth/login')
        .send({ email: 'user@test.com', password: 'User123!' });

      userToken = userLogin.body.accessToken;
    });

    it('admin should access admin endpoint', async () => {
      await request(app.getHttpServer())
        .get('/admin/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .expect(200);
    });

    it('user should not access admin endpoint', async () => {
      await request(app.getHttpServer())
        .get('/admin/users')
        .set('Authorization', `Bearer ${userToken}`)
        .expect(403);
    });
  });
});
```

## Scenario 2: Complex Business Flow

### Order Workflow

```typescript
describe('Order Processing E2E', () => {
  let userRepository: UserRepository;
  let productRepository: ProductRepository;
  let orderRepository: OrderRepository;

  describe('Complete order lifecycle', () => {
    it('should process order from checkout to delivery', async () => {
      // 1. Create user
      const userRes = await request(app.getHttpServer())
        .post('/users')
        .send({
          email: 'customer@test.com',
          name: 'Customer',
          age: 30,
        });

      const userId = userRes.body.id;

      // 2. Login
      const loginRes = await request(app.getHttpServer())
        .post('/auth/login')
        .send({ email: 'customer@test.com', password: 'password' });

      const token = loginRes.body.accessToken;

      // 3. Browse products (verify they exist)
      const productsRes = await request(app.getHttpServer())
        .get('/products')
        .expect(200);

      expect(productsRes.body.length).toBeGreaterThan(0);
      const productId = productsRes.body[0].id;

      // 4. Add to cart
      const cartRes = await request(app.getHttpServer())
        .post('/cart/items')
        .set('Authorization', `Bearer ${token}`)
        .send({ productId, quantity: 2 })
        .expect(201);

      expect(cartRes.body.items).toHaveLength(1);

      // 5. Checkout
      const orderRes = await request(app.getHttpServer())
        .post('/orders/checkout')
        .set('Authorization', `Bearer ${token}`)
        .send({
          shippingAddress: '123 Main St',
          paymentMethod: 'credit_card',
        })
        .expect(201);

      const orderId = orderRes.body.id;
      expect(orderRes.body.status).toBe('PENDING_PAYMENT');

      // 6. Process payment
      const paymentRes = await request(app.getHttpServer())
        .post(`/orders/${orderId}/pay`)
        .set('Authorization', `Bearer ${token}`)
        .send({
          amount: orderRes.body.total,
          cardToken: 'tok_visa',
        })
        .expect(200);

      expect(paymentRes.body.status).toBe('PAID');

      // 7. Verify in database
      const order = await orderRepository.findById(orderId);
      expect(order.status).toBe('PAID');
      expect(order.userId).toBe(userId);

      // 8. Check inventory reduced
      const product = await productRepository.findById(productId);
      expect(product.stock).toBeLessThan(productsRes.body[0].stock);
    });

    it('should handle payment failure gracefully', async () => {
      // Create user and order...
      const userRes = await request(app.getHttpServer())
        .post('/users')
        .send({ email: 'poor@test.com', name: 'Poor', age: 30 });

      const loginRes = await request(app.getHttpServer())
        .post('/auth/login')
        .send({ email: 'poor@test.com', password: 'password' });

      const token = loginRes.body.accessToken;

      // Create order
      const orderRes = await request(app.getHttpServer())
        .post('/orders/checkout')
        .set('Authorization', `Bearer ${token}`)
        .send({
          shippingAddress: '123 Main St',
          paymentMethod: 'credit_card',
        })
        .expect(201);

      // Try to pay with declined card
      await request(app.getHttpServer())
        .post(`/orders/${orderRes.body.id}/pay`)
        .set('Authorization', `Bearer ${token}`)
        .send({
          amount: orderRes.body.total,
          cardToken: 'tok_chargeDeclined',
        })
        .expect(402); // Payment required

      // Verify order still pending
      const order = await orderRepository.findById(orderRes.body.id);
      expect(order.status).toBe('PENDING_PAYMENT');

      // Inventory should not be reduced
      const product = await productRepository.findById(orderRes.body.items[0].productId);
      expect(product.stock).toBe(100); // Original amount
    });
  });
});
```

## Scenario 3: Concurrent Operations

### Race Condition Testing

```typescript
describe('Concurrent Operations E2E', () => {
  it('should handle concurrent user creation', async () => {
    // Simulate 10 concurrent requests
    const promises = Array.from({ length: 10 }).map((_, i) =>
      request(app.getHttpServer())
        .post('/users')
        .send({
          email: `user${i}@test.com`,
          name: `User ${i}`,
          age: 20 + i,
        })
    );

    const results = await Promise.all(promises);

    // All should succeed
    results.forEach(res => {
      expect(res.status).toBe(201);
    });

    // All should have unique IDs
    const ids = results.map(r => r.body.id);
    expect(new Set(ids).size).toBe(10);

    // All should be in database
    const users = await userRepository.find();
    expect(users).toHaveLength(10);
  });

  it('should handle concurrent updates to same resource', async () => {
    // Create user
    const createRes = await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'user@test.com', name: 'User', age: 30 });

    const userId = createRes.body.id;

    // Simulate 5 concurrent updates
    const promises = Array.from({ length: 5 }).map((_, i) =>
      request(app.getHttpServer())
        .put(`/users/${userId}`)
        .send({ age: 30 + i })
    );

    const results = await Promise.all(promises);

    // All should complete (some may succeed, some may have conflicts)
    expect(results.length).toBe(5);

    // Final state should be consistent
    const user = await userRepository.findById(userId);
    expect(user).toBeDefined();
    expect(user.age).toBeGreaterThan(30);
  });

  it('should prevent duplicate email with concurrent requests', async () => {
    // Simulate 3 concurrent registrations with same email
    const promises = Array.from({ length: 3 }).map(() =>
      request(app.getHttpServer())
        .post('/users')
        .send({
          email: 'test@test.com',
          name: 'Test',
          age: 30,
        })
    );

    const results = await Promise.all(promises);

    // Only one should succeed
    const successful = results.filter(r => r.status === 201);
    expect(successful).toHaveLength(1);

    // Others should get conflict
    const conflicts = results.filter(r => r.status === 409);
    expect(conflicts.length).toBeGreaterThan(0);

    // Only one user in database
    const users = await userRepository.find({ email: 'test@test.com' });
    expect(users).toHaveLength(1);
  });
});
```

## Scenario 4: Error Scenario Testing

### Validation Errors

```typescript
describe('Input Validation E2E', () => {
  it('should reject invalid email', async () => {
    await request(app.getHttpServer())
      .post('/users')
      .send({
        email: 'not-an-email',
        name: 'User',
        age: 30,
      })
      .expect(400)
      .expect(res => {
        expect(res.body.message).toContain('email');
      });
  });

  it('should reject negative age', async () => {
    await request(app.getHttpServer())
      .post('/users')
      .send({
        email: 'user@test.com',
        name: 'User',
        age: -5,
      })
      .expect(400);
  });

  it('should reject missing required field', async () => {
    await request(app.getHttpServer())
      .post('/users')
      .send({
        email: 'user@test.com',
        // name is missing
        age: 30,
      })
      .expect(400);
  });

  it('should truncate long names', async () => {
    const longName = 'a'.repeat(300);

    const res = await request(app.getHttpServer())
      .post('/users')
      .send({
        email: 'user@test.com',
        name: longName,
        age: 30,
      });

    // May fail validation or truncate
    if (res.status === 201) {
      expect(res.body.name.length).toBeLessThanOrEqual(255);
    } else {
      expect(res.status).toBe(400);
    }
  });
});
```

## Scenario 5: State Transition Testing

### Workflow Status Transitions

```typescript
describe('Order Status Transitions', () => {
  it('should follow valid status transitions', async () => {
    // Create order (PENDING)
    const orderRes = await request(app.getHttpServer())
      .post('/orders')
      .send({ /* ... */ })
      .expect(201);

    const orderId = orderRes.body.id;
    expect(orderRes.body.status).toBe('PENDING');

    // Transition to PROCESSING
    const processingRes = await request(app.getHttpServer())
      .patch(`/orders/${orderId}/status`)
      .send({ status: 'PROCESSING' })
      .expect(200);

    expect(processingRes.body.status).toBe('PROCESSING');

    // Transition to SHIPPED
    const shippedRes = await request(app.getHttpServer())
      .patch(`/orders/${orderId}/status`)
      .send({ status: 'SHIPPED' })
      .expect(200);

    expect(shippedRes.body.status).toBe('SHIPPED');

    // Transition to DELIVERED
    const deliveredRes = await request(app.getHttpServer())
      .patch(`/orders/${orderId}/status`)
      .send({ status: 'DELIVERED' })
      .expect(200);

    expect(deliveredRes.body.status).toBe('DELIVERED');
  });

  it('should reject invalid status transitions', async () => {
    // Create order (PENDING)
    const orderRes = await request(app.getHttpServer())
      .post('/orders')
      .send({ /* ... */ })
      .expect(201);

    // Try invalid transition: PENDING → DELIVERED (skip PROCESSING, SHIPPED)
    await request(app.getHttpServer())
      .patch(`/orders/${orderRes.body.id}/status`)
      .send({ status: 'DELIVERED' })
      .expect(400)
      .expect(res => {
        expect(res.body.message).toContain('invalid transition');
      });
  });

  it('should reject backward transitions', async () => {
    // Create and process order
    const orderRes = await request(app.getHttpServer())
      .post('/orders')
      .send({ /* ... */ })
      .expect(201);

    const orderId = orderRes.body.id;

    // Move forward
    await request(app.getHttpServer())
      .patch(`/orders/${orderId}/status`)
      .send({ status: 'PROCESSING' })
      .expect(200);

    // Try to go backward: PROCESSING → PENDING
    await request(app.getHttpServer())
      .patch(`/orders/${orderId}/status`)
      .send({ status: 'PENDING' })
      .expect(400);
  });
});
```
