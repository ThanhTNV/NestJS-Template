# Advanced Mocking Patterns

This document covers sophisticated mocking techniques for complex testing scenarios.

## Mock Object Strategies

### Partial Mocks (Real + Mocked Methods)

When you want to test a class but mock only specific dependencies:

```typescript
// user.service.ts
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly emailService: EmailService,
    private readonly configService: ConfigService,
  ) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    // ...
  }
}

// user.service.spec.ts - Mock only repository, use real email service
describe('UserService - With Real EmailService', () => {
  let service: UserService;

  const mockUserRepository = {
    findByEmail: jest.fn(),
    save: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: UserRepository,
          useValue: mockUserRepository,
        },
        {
          provide: EmailService,
          useValue: new EmailService(/* dependencies */),
        },
        {
          provide: ConfigService,
          useValue: { get: jest.fn().mockReturnValue('test') },
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
  });
});
```

### Deep Mocking (Nested Dependencies)

When mocks have nested properties or methods:

```typescript
// Service depends on config with nested properties
@Injectable()
export class PaymentService {
  constructor(private readonly configService: ConfigService) {}

  async processPayment(amount: number): Promise<void> {
    const key = this.configService.get('stripe.apiKey');
    const maxRetries = this.configService.get('payment.maxRetries');
    // ...
  }
}

// Deep mock setup
const mockConfigService = {
  get: jest.fn((key: string) => {
    const config = {
      'stripe.apiKey': 'sk_test_123',
      'payment.maxRetries': 3,
      'payment.timeout': 5000,
    };
    return config[key];
  }),
};

const module = await Test.createTestingModule({
  providers: [
    PaymentService,
    {
      provide: ConfigService,
      useValue: mockConfigService,
    },
  ],
}).compile();
```

### Spying on Real Implementations

Test real instances but track calls:

```typescript
// Test with real service but spy on methods
describe('UserService - Spied', () => {
  let service: UserService;
  let repository: UserRepository;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        UserRepository,
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    repository = module.get<UserRepository>(UserRepository);
  });

  it('should call repository.save with correct data', async () => {
    // Spy on real method
    const saveSpy = jest.spyOn(repository, 'save');
    const dto = { email: 'test@test.com', name: 'John', age: 30 };

    await service.createUser(dto);

    expect(saveSpy).toHaveBeenCalledWith(expect.objectContaining({
      email: dto.email,
    }));

    saveSpy.mockRestore();
  });
});
```

## Mock Return Values

### Basic Returns

```typescript
const mockRepository = {
  findById: jest.fn().mockResolvedValue({ id: '1', name: 'John' }),
  save: jest.fn().mockResolvedValue({ id: '1', name: 'John' }),
  delete: jest.fn().mockResolvedValue(undefined),
};
```

### Conditional Returns

```typescript
const mockRepository = {
  findById: jest.fn((id: string) => {
    if (id === '1') {
      return Promise.resolve({ id: '1', name: 'John' });
    }
    return Promise.resolve(null);
  }),
};

// Usage in test
expect(await repository.findById('1')).toEqual({ id: '1', name: 'John' });
expect(await repository.findById('999')).toBeNull();
```

### Different Values Per Call

```typescript
const mockRepository = {
  find: jest.fn()
    .mockResolvedValueOnce([{ id: '1' }])  // First call
    .mockResolvedValueOnce([])               // Second call
    .mockResolvedValueOnce([{ id: '2' }]),  // Third call
};

// First call returns one user
expect(await repository.find()).toHaveLength(1);

// Second call returns empty
expect(await repository.find()).toHaveLength(0);

// Third call returns different user
expect(await repository.find()).toHaveLength(1);
```

### Rejecting Promises

```typescript
const mockRepository = {
  findById: jest.fn().mockRejectedValue(new DatabaseError('Connection failed')),
  save: jest.fn().mockRejectedValueOnce(new Error('Timeout')),
};
```

## Verifying Mock Calls

### Basic Call Verification

```typescript
const mockRepository = { save: jest.fn() };

// Called exactly once
expect(mockRepository.save).toHaveBeenCalledTimes(1);

// Called with specific arguments
expect(mockRepository.save).toHaveBeenCalledWith(user);

// Called with arguments matching partial object
expect(mockRepository.save).toHaveBeenCalledWith(
  expect.objectContaining({ email: 'test@test.com' })
);

// Not called
expect(mockRepository.save).not.toHaveBeenCalled();
```

### Call Argument Matching

```typescript
const mockService = { process: jest.fn() };

await service.doSomething(data);

// Verify with matchers
expect(mockService.process).toHaveBeenCalledWith(
  expect.any(Object),                        // Any object
  expect.any(String),                        // Any string
  expect.stringMatching(/^test/),            // String pattern
  expect.arrayContaining(['value']),         // Array contains
  expect.objectContaining({ key: 'value' }) // Object contains
);
```

### Call Order Verification

```typescript
const mockRepository = { findById: jest.fn(), save: jest.fn() };

await service.createAndSave(data);

// Verify order of calls
expect(mockRepository.findById).toHaveBeenCalledBefore(mockRepository.save);

// Or check call order in sequence
const calls = jest.mock.calls;
expect(mockRepository.findById).toHaveBeenNthCalledWith(1, 'id');
expect(mockRepository.save).toHaveBeenNthCalledWith(1, expect.any(Object));
```

## Dynamic Mock Setup

### Factory Pattern

```typescript
function createMockUserRepository(overrides = {}) {
  return {
    findById: jest.fn(),
    findByEmail: jest.fn(),
    save: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
    ...overrides,
  };
}

// Usage
const mockRepo = createMockUserRepository({
  findById: jest.fn().mockResolvedValue({ id: '1' }),
});
```

### Builder Pattern

```typescript
class MockUserRepositoryBuilder {
  private mock = {
    findById: jest.fn(),
    save: jest.fn(),
    delete: jest.fn(),
  };

  withFindById(user: User) {
    this.mock.findById.mockResolvedValue(user);
    return this;
  }

  withSaveError(error: Error) {
    this.mock.save.mockRejectedValue(error);
    return this;
  }

  build() {
    return this.mock;
  }
}

// Usage
const mockRepo = new MockUserRepositoryBuilder()
  .withFindById({ id: '1', name: 'John' })
  .withSaveError(new Error('DB Error'))
  .build();
```

## Testing with Multiple Mocks

### Service with Multiple Dependencies

```typescript
describe('OrderService', () => {
  let service: OrderService;

  const mockOrderRepository = {
    save: jest.fn(),
    findById: jest.fn(),
  };

  const mockProductService = {
    getProduct: jest.fn(),
    decreaseStock: jest.fn(),
  };

  const mockPaymentService = {
    processPayment: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        OrderService,
        {
          provide: OrderRepository,
          useValue: mockOrderRepository,
        },
        {
          provide: ProductService,
          useValue: mockProductService,
        },
        {
          provide: PaymentService,
          useValue: mockPaymentService,
        },
      ],
    }).compile();

    service = module.get<OrderService>(OrderService);
    jest.clearAllMocks();
  });

  it('should create order with all dependencies', async () => {
    const productId = '1';
    const quantity = 2;
    const amount = 100;

    // Arrange
    mockProductService.getProduct.mockResolvedValue({
      id: productId,
      name: 'Product',
      price: 50,
      stock: 10,
    });

    mockPaymentService.processPayment.mockResolvedValue({
      transactionId: 'txn_123',
    });

    mockOrderRepository.save.mockResolvedValue({
      id: '1',
      productId,
      quantity,
      amount,
    });

    // Act
    const result = await service.createOrder({ productId, quantity });

    // Assert
    expect(mockProductService.getProduct).toHaveBeenCalledWith(productId);
    expect(mockPaymentService.processPayment).toHaveBeenCalledWith(amount);
    expect(mockOrderRepository.save).toHaveBeenCalled();
    expect(result.id).toBe('1');
  });
});
```

## Mocking Async/Await Patterns

### Promise Resolution

```typescript
const mockService = {
  // Implicit promise wrapping
  method: jest.fn().mockResolvedValue(data),
};

// Equivalent to:
const mockService = {
  method: jest.fn(() => Promise.resolve(data)),
};
```

### Async Function Mocks

```typescript
const mockService = {
  // Mock async function
  asyncMethod: jest.fn(async (id: string) => {
    return { id, data: 'test' };
  }),
};

it('should handle async', async () => {
  const result = await mockService.asyncMethod('1');
  expect(result).toEqual({ id: '1', data: 'test' });
});
```

### Race Condition Testing

```typescript
it('should handle concurrent calls', async () => {
  const mockRepository = {
    save: jest.fn()
      .mockResolvedValueOnce({ id: '1' }) // First concurrent call
      .mockResolvedValueOnce({ id: '2' }), // Second concurrent call
  };

  const [result1, result2] = await Promise.all([
    service.createUser({ email: 'a@test.com', name: 'A', age: 30 }),
    service.createUser({ email: 'b@test.com', name: 'B', age: 25 }),
  ]);

  expect(result1.id).toBe('1');
  expect(result2.id).toBe('2');
  expect(mockRepository.save).toHaveBeenCalledTimes(2);
});
```

## Auto-Mocking Techniques

### Jest Auto Mock

```typescript
jest.mock('./user.repository');

import { UserRepository } from './user.repository';

// All methods auto-mocked as jest.fn()
const mockRepository = UserRepository as jest.Mocked<typeof UserRepository>;

mockRepository.findById.mockResolvedValue({ id: '1' });
```

### Manual Auto Mock

```typescript
function createAutoMock<T>(obj: T): jest.Mocked<T> {
  const mock = {} as jest.Mocked<T>;
  
  for (const key in obj) {
    if (typeof obj[key] === 'function') {
      mock[key] = jest.fn();
    }
  }
  
  return mock;
}

// Usage
const mockRepository = createAutoMock(userRepository);
```

## Common Mocking Mistakes

### ❌ Mistake 1: Not Clearing Mocks Between Tests

```typescript
// BAD
describe('Service', () => {
  const mockRepository = { save: jest.fn() };

  it('test 1', async () => {
    await service.create(data);
    expect(mockRepository.save).toHaveBeenCalledTimes(1);
  });

  it('test 2', async () => {
    await service.create(data);
    // FAILS - mock still has 1 call from test 1
    expect(mockRepository.save).toHaveBeenCalledTimes(1);
  });
});

// GOOD
describe('Service', () => {
  const mockRepository = { save: jest.fn() };

  beforeEach(() => {
    jest.clearAllMocks(); // Clear before each test
  });

  it('test 1', async () => { /* ... */ });
  it('test 2', async () => { /* ... */ });
});
```

### ❌ Mistake 2: Over-Mocking (Mocking Everything)

```typescript
// BAD - Even mock simple value objects
const mockUser = {
  getName: jest.fn().mockReturnValue('John'),
  getEmail: jest.fn().mockReturnValue('test@test.com'),
};

// GOOD - Only mock external dependencies
const user = {
  name: 'John',
  email: 'test@test.com',
};

const mockRepository = {
  save: jest.fn(),
};
```

### ❌ Mistake 3: Mocks Return Wrong Type

```typescript
// BAD - Repository returns object, mock returns undefined
const mockRepository = {
  findById: jest.fn(), // Returns undefined by default
};

it('should get user', async () => {
  const result = await service.getUser('1');
  expect(result).toEqual(expect.anything()); // Vague assertion
});

// GOOD - Mock returns correct type
const mockRepository = {
  findById: jest.fn().mockResolvedValue({ id: '1', name: 'John' }),
};

it('should get user', async () => {
  const result = await service.getUser('1');
  expect(result).toEqual({ id: '1', name: 'John' }); // Clear assertion
});
```

### ❌ Mistake 4: Testing Mock Behavior Instead of Real Code

```typescript
// BAD - Testing that mock was called, not that business logic works
it('should save user', async () => {
  mockRepository.save.mockResolvedValue({ id: '1' });

  await service.createUser(dto);

  expect(mockRepository.save).toHaveBeenCalled(); // Only checks mock
});

// GOOD - Test business logic outcome
it('should save user', async () => {
  mockRepository.save.mockResolvedValue({ id: '1' });
  mockRepository.findByEmail.mockResolvedValue(null);

  const result = await service.createUser(dto);

  expect(result.id).toBe('1'); // Test actual result
  expect(mockRepository.findByEmail).toHaveBeenCalledWith(dto.email); // Verify correct method called
});
```
