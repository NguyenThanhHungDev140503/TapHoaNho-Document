# Lập Trình Bất Đồng Bộ trong TypeScript

## Mục Lục

- [Junior Level - Cơ Bản](#junior-level---cơ-bản)
  - [1. Giới Thiệu về Asynchronous Programming](#1-giới-thiệu-về-asynchronous-programming)
  - [2. Callbacks và Callback Hell](#2-callbacks-và-callback-hell)
  - [3. Promises Cơ Bản](#3-promises-cơ-bản)
  - [4. Async/Await Syntax](#4-asyncawait-syntax)
  - [5. Error Handling](#5-error-handling)
  - [6. Common Pitfalls](#6-common-pitfalls)

- [Middle Level - Trung Cấp](#middle-level---trung-cấp)
  - [1. Promise Chaining và Composition](#1-promise-chaining-và-composition)
  - [2. Promise Combinators](#2-promise-combinators)
  - [3. Async/Await với Loops](#3-asyncawait-với-loops)
  - [4. Parallel vs Sequential Execution](#4-parallel-vs-sequential-execution)
  - [5. Error Handling Nâng Cao](#5-error-handling-nâng-cao)
  - [6. Timeout và Cancellation](#6-timeout-và-cancellation)

- [Senior Level](#senior-level) - Xem file `Advanced-Async-Patterns.md`
- [Principal Level](#principal-level) - Xem file `Principal-Async-Patterns.md`

---

## Junior Level - Cơ Bản

### 1. Giới Thiệu về Asynchronous Programming

#### Synchronous vs Asynchronous

**Synchronous (Đồng bộ)**: Code chạy tuần tự, mỗi dòng phải đợi dòng trước hoàn thành.

```typescript
// Synchronous code
console.log('Start');
const result = heavyComputation(); // Block cho đến khi xong
console.log('Result:', result);
console.log('End');

// Output:
// Start
// Result: ...
// End
```

**Asynchronous (Bất đồng bộ)**: Code không block, cho phép các operations khác chạy trong khi đợi.

```typescript
// Asynchronous code
console.log('Start');
fetchData().then(result => {
  console.log('Result:', result);
});
console.log('End');

// Output:
// Start
// End
// Result: ...
```

#### Tại Sao Cần Async Programming?

1. **Non-blocking I/O**: Không block main thread khi đợi network requests, file operations
2. **Better User Experience**: UI không bị freeze khi thực hiện long-running tasks
3. **Efficient Resource Usage**: CPU có thể làm việc khác trong khi đợi I/O
4. **Scalability**: Server có thể handle nhiều requests đồng thời

#### Use Cases Phổ Biến

```typescript
// 1. API Calls
async function fetchUser(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);
  return response.json();
}

// 2. Database Operations
async function saveUser(user: User): Promise<void> {
  await db.users.insert(user);
}

// 3. File Operations
async function readFile(path: string): Promise<string> {
  return fs.promises.readFile(path, 'utf-8');
}

// 4. Timers
async function delay(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

### 2. Callbacks và Callback Hell

#### Callbacks Cơ Bản

Callback là function được pass làm argument và được gọi sau khi operation hoàn thành.

```typescript
// Example 1: Simple callback
function fetchData(callback: (data: string) => void): void {
  setTimeout(() => {
    callback('Data loaded');
  }, 1000);
}

// Usage
fetchData((data) => {
  console.log(data); // "Data loaded" sau 1 giây
});
```

**Giải thích:**
- `fetchData` nhận một callback function
- Sau 1 giây, callback được gọi với data
- Pattern này được dùng rộng rãi trong Node.js cũ

#### Error-First Callbacks (Node.js Convention)

```typescript
// Example 2: Error-first callback pattern
function readFileCallback(
  path: string,
  callback: (error: Error | null, data?: string) => void
): void {
  // Simulate file reading
  setTimeout(() => {
    if (path === '') {
      callback(new Error('Path cannot be empty'));
    } else {
      callback(null, 'File content');
    }
  }, 1000);
}

// Usage
readFileCallback('file.txt', (error, data) => {
  if (error) {
    console.error('Error:', error.message);
    return;
  }
  console.log('Data:', data);
});
```

**Giải thích:**
- Parameter đầu tiên luôn là `error` (null nếu success)
- Parameter thứ hai là `data` (undefined nếu error)
- Pattern này giúp consistent error handling

#### Callback Hell (Pyramid of Doom)

```typescript
// Example 3: Callback hell - ❌ BAD PRACTICE
function processOrder(orderId: string): void {
  getOrder(orderId, (error1, order) => {
    if (error1) {
      console.error(error1);
      return;
    }
    
    getUser(order.userId, (error2, user) => {
      if (error2) {
        console.error(error2);
        return;
      }
      
      getPayment(order.paymentId, (error3, payment) => {
        if (error3) {
          console.error(error3);
          return;
        }
        
        sendEmail(user.email, order, payment, (error4) => {
          if (error4) {
            console.error(error4);
            return;
          }
          
          console.log('Order processed successfully');
        });
      });
    });
  });
}
```

**Vấn Đề với Callback Hell:**
1. **Khó đọc**: Code lồng nhau nhiều tầng
2. **Khó maintain**: Thêm/sửa logic rất phức tạp
3. **Error handling**: Phải check error ở mỗi level
4. **Khó debug**: Stack trace khó hiểu

**Giải pháp**: Sử dụng Promises hoặc async/await (sẽ học ở phần sau)

### 3. Promises Cơ Bản

#### Promise là gì?

Promise là object đại diện cho kết quả của một async operation trong tương lai. Promise có 3 states:
- **Pending**: Đang chờ kết quả
- **Fulfilled**: Thành công, có value
- **Rejected**: Thất bại, có error

```typescript
// Example 1: Tạo Promise cơ bản
function fetchData(): Promise<string> {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      const success = Math.random() > 0.5;

      if (success) {
        resolve('Data loaded successfully'); // Fulfilled
      } else {
        reject(new Error('Failed to load data')); // Rejected
      }
    }, 1000);
  });
}
```

**Giải thích:**
- `new Promise()` nhận executor function với 2 parameters: `resolve` và `reject`
- `resolve(value)`: Chuyển Promise sang Fulfilled state
- `reject(error)`: Chuyển Promise sang Rejected state
- Promise chỉ có thể settle (fulfilled hoặc rejected) một lần

#### Consuming Promises với .then() và .catch()

```typescript
// Example 2: Sử dụng .then() và .catch()
fetchData()
  .then((data) => {
    // Chạy khi Promise fulfilled
    console.log('Success:', data);
    return data.toUpperCase(); // Return value cho .then() tiếp theo
  })
  .catch((error) => {
    // Chạy khi Promise rejected
    console.error('Error:', error.message);
  })
  .finally(() => {
    // Luôn chạy, dù success hay error
    console.log('Cleanup or logging');
  });
```

**Giải thích:**
- `.then(onFulfilled)`: Handle success case
- `.catch(onRejected)`: Handle error case
- `.finally(onFinally)`: Cleanup logic, chạy trong mọi trường hợp
- `.then()` return một Promise mới, cho phép chaining

#### Promise Chaining

```typescript
// Example 3: Promise chaining
interface User {
  id: string;
  name: string;
  email: string;
}

interface Order {
  id: string;
  userId: string;
  total: number;
}

function getUser(userId: string): Promise<User> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve({ id: userId, name: 'John Doe', email: 'john@example.com' });
    }, 500);
  });
}

function getOrders(userId: string): Promise<Order[]> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve([
        { id: '1', userId, total: 100 },
        { id: '2', userId, total: 200 },
      ]);
    }, 500);
  });
}

// Chaining promises
getUser('user123')
  .then((user) => {
    console.log('User:', user.name);
    return getOrders(user.id); // Return Promise
  })
  .then((orders) => {
    console.log('Orders:', orders.length);
    const total = orders.reduce((sum, order) => sum + order.total, 0);
    return total;
  })
  .then((total) => {
    console.log('Total:', total);
  })
  .catch((error) => {
    console.error('Error:', error);
  });
```

**Giải thích:**
- Mỗi `.then()` return một Promise mới
- Value return từ `.then()` trở thành resolved value của Promise tiếp theo
- Nếu return Promise, Promise đó được "unwrapped"
- Error propagate xuống `.catch()` gần nhất

#### Tạo Resolved/Rejected Promises

```typescript
// Example 4: Promise.resolve() và Promise.reject()

// Tạo Promise đã resolved
const resolvedPromise = Promise.resolve('Immediate value');
resolvedPromise.then(value => console.log(value)); // "Immediate value"

// Tạo Promise đã rejected
const rejectedPromise = Promise.reject(new Error('Immediate error'));
rejectedPromise.catch(error => console.error(error.message)); // "Immediate error"

// Use case: Wrap synchronous value
function getUserFromCache(userId: string): Promise<User | null> {
  const cached = cache.get(userId);
  if (cached) {
    return Promise.resolve(cached); // Return Promise ngay lập tức
  }
  return fetchUserFromAPI(userId); // Return Promise từ API
}
```

### 4. Async/Await Syntax

#### Async Functions

`async` keyword biến function thành async function, luôn return Promise.

```typescript
// Example 1: Async function cơ bản
async function fetchUser(userId: string): Promise<User> {
  // Function body có thể dùng await
  const response = await fetch(`/api/users/${userId}`);
  const user = await response.json();
  return user; // Tự động wrap trong Promise
}

// Tương đương với:
function fetchUserPromise(userId: string): Promise<User> {
  return fetch(`/api/users/${userId}`)
    .then(response => response.json());
}
```

**Giải thích:**
- `async` function luôn return Promise
- Nếu return value, value được wrap trong `Promise.resolve(value)`
- Nếu throw error, error được wrap trong `Promise.reject(error)`

#### Await Keyword

`await` pause execution của async function cho đến khi Promise resolve.

```typescript
// Example 2: Await với error handling
async function processOrder(orderId: string): Promise<void> {
  try {
    // Await pause execution cho đến khi Promise resolve
    const order = await getOrder(orderId);
    console.log('Order:', order);

    const user = await getUser(order.userId);
    console.log('User:', user);

    const payment = await getPayment(order.paymentId);
    console.log('Payment:', payment);

    await sendEmail(user.email, order, payment);
    console.log('Email sent');
  } catch (error) {
    console.error('Error processing order:', error);
    throw error; // Re-throw nếu cần
  }
}
```

**Giải thích:**
- `await` chỉ dùng được trong `async` function
- `await` "unwrap" Promise, return resolved value
- Nếu Promise rejected, `await` throw error
- Code đọc như synchronous code, dễ hiểu hơn

#### So Sánh: Promises vs Async/Await

```typescript
// Example 3: Cùng logic, 2 cách viết

// ❌ Với Promises (verbose)
function getUserDataPromise(userId: string): Promise<string> {
  return getUser(userId)
    .then((user) => {
      return getOrders(user.id);
    })
    .then((orders) => {
      const total = orders.reduce((sum, o) => sum + o.total, 0);
      return `User has ${orders.length} orders, total: $${total}`;
    })
    .catch((error) => {
      console.error('Error:', error);
      throw error;
    });
}

// ✅ Với Async/Await (clean)
async function getUserDataAsync(userId: string): Promise<string> {
  try {
    const user = await getUser(userId);
    const orders = await getOrders(user.id);
    const total = orders.reduce((sum, o) => sum + o.total, 0);
    return `User has ${orders.length} orders, total: $${total}`;
  } catch (error) {
    console.error('Error:', error);
    throw error;
  }
}
```

**Ưu điểm của Async/Await:**
- Code đọc như synchronous, dễ hiểu
- Error handling với try/catch quen thuộc
- Debugging dễ dàng hơn
- Ít boilerplate code hơn

### 5. Error Handling

#### Try/Catch với Async/Await

```typescript
// Example 1: Basic error handling
async function fetchUserSafely(userId: string): Promise<User | null> {
  try {
    const response = await fetch(`/api/users/${userId}`);

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const user = await response.json();
    return user;
  } catch (error) {
    console.error('Failed to fetch user:', error);
    return null; // Return fallback value
  }
}
```

**Giải thích:**
- `try` block chứa async operations
- `catch` block handle tất cả errors (từ await hoặc throw)
- Có thể return fallback value hoặc re-throw error

#### Multiple Try/Catch Blocks

```typescript
// Example 2: Granular error handling
async function processUserOrder(userId: string, orderId: string): Promise<void> {
  let user: User | null = null;
  let order: Order | null = null;

  // Try 1: Fetch user
  try {
    user = await getUser(userId);
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw new Error('User not found');
  }

  // Try 2: Fetch order
  try {
    order = await getOrder(orderId);
  } catch (error) {
    console.error('Failed to fetch order:', error);
    throw new Error('Order not found');
  }

  // Try 3: Process payment
  try {
    await processPayment(user, order);
  } catch (error) {
    console.error('Payment failed:', error);
    // Có thể retry hoặc notify user
    throw new Error('Payment processing failed');
  }
}
```

#### Custom Error Types

```typescript
// Example 3: Custom error classes
class NetworkError extends Error {
  constructor(message: string, public statusCode: number) {
    super(message);
    this.name = 'NetworkError';
  }
}

class ValidationError extends Error {
  constructor(message: string, public field: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

async function createUser(userData: any): Promise<User> {
  // Validation
  if (!userData.email) {
    throw new ValidationError('Email is required', 'email');
  }

  try {
    const response = await fetch('/api/users', {
      method: 'POST',
      body: JSON.stringify(userData),
    });

    if (!response.ok) {
      throw new NetworkError('Failed to create user', response.status);
    }

    return await response.json();
  } catch (error) {
    if (error instanceof NetworkError) {
      console.error('Network error:', error.statusCode);
    } else if (error instanceof ValidationError) {
      console.error('Validation error:', error.field);
    } else {
      console.error('Unknown error:', error);
    }
    throw error;
  }
}
```

### 6. Common Pitfalls

#### ❌ Pitfall 1: Quên await

```typescript
// ❌ SAI: Quên await
async function badExample() {
  const user = getUser('123'); // Trả về Promise, không phải User!
  console.log(user.name); // Error: Cannot read property 'name' of Promise
}

// ✅ ĐÚNG: Có await
async function goodExample() {
  const user = await getUser('123'); // Đợi Promise resolve
  console.log(user.name); // OK
}
```

#### ❌ Pitfall 2: Không handle errors

```typescript
// ❌ SAI: Không có error handling
async function badFetch() {
  const data = await fetch('/api/data'); // Nếu fail, unhandled rejection!
  return data.json();
}

// ✅ ĐÚNG: Có try/catch
async function goodFetch() {
  try {
    const data = await fetch('/api/data');
    return await data.json();
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error;
  }
}
```

#### ❌ Pitfall 3: Async function trong forEach

```typescript
// ❌ SAI: forEach không đợi async functions
async function badLoop(userIds: string[]) {
  userIds.forEach(async (id) => {
    const user = await getUser(id); // Không đợi!
    console.log(user);
  });
  console.log('Done'); // Chạy trước khi users được fetch
}

// ✅ ĐÚNG: Dùng for...of
async function goodLoop(userIds: string[]) {
  for (const id of userIds) {
    const user = await getUser(id); // Đợi từng user
    console.log(user);
  }
  console.log('Done'); // Chạy sau khi tất cả users được fetch
}

// ✅ ĐÚNG: Dùng Promise.all nếu muốn parallel
async function parallelLoop(userIds: string[]) {
  const users = await Promise.all(
    userIds.map(id => getUser(id))
  );
  users.forEach(user => console.log(user));
  console.log('Done');
}
```

#### ❌ Pitfall 4: Return trong try/catch

```typescript
// ❌ SAI: Finally không chạy nếu return trong try
async function badFinally() {
  try {
    const data = await fetchData();
    return data; // Return ngay, finally có chạy không?
  } catch (error) {
    console.error(error);
  } finally {
    console.log('Cleanup'); // Vẫn chạy! Nhưng dễ nhầm
  }
}

// ✅ ĐÚNG: Rõ ràng hơn
async function goodFinally() {
  let data;
  try {
    data = await fetchData();
  } catch (error) {
    console.error(error);
    throw error;
  } finally {
    console.log('Cleanup'); // Luôn chạy
  }
  return data;
}
```

#### ❌ Pitfall 5: Floating Promises

```typescript
// ❌ SAI: Promise không được await hoặc handle
async function badFloating() {
  saveToDatabase(data); // Floating promise! Không biết khi nào xong
  console.log('Saved'); // Có thể chạy trước khi save xong
}

// ✅ ĐÚNG: Await hoặc handle promise
async function goodAwaited() {
  await saveToDatabase(data);
  console.log('Saved'); // Chắc chắn save xong
}

// ✅ ĐÚNG: Nếu không cần đợi, handle error
async function goodHandled() {
  saveToDatabase(data).catch(error => {
    console.error('Save failed:', error);
  });
  console.log('Save initiated');
}
```

---

## Middle Level - Trung Cấp

### 1. Promise Chaining và Composition

#### Advanced Promise Chaining

```typescript
// Example 1: Complex promise chain với transformation
interface RawData {
  id: string;
  value: number;
}

interface ProcessedData {
  id: string;
  value: number;
  doubled: number;
  category: string;
}

function fetchRawData(): Promise<RawData[]> {
  return fetch('/api/data').then(res => res.json());
}

function processData(data: RawData[]): Promise<ProcessedData[]> {
  return new Promise((resolve) => {
    setTimeout(() => {
      const processed = data.map(item => ({
        ...item,
        doubled: item.value * 2,
        category: item.value > 50 ? 'high' : 'low',
      }));
      resolve(processed);
    }, 100);
  });
}

function saveData(data: ProcessedData[]): Promise<void> {
  return fetch('/api/save', {
    method: 'POST',
    body: JSON.stringify(data),
  }).then(() => undefined);
}

// Chaining với transformation
fetchRawData()
  .then(rawData => {
    console.log('Fetched:', rawData.length, 'items');
    return processData(rawData);
  })
  .then(processedData => {
    console.log('Processed:', processedData.length, 'items');
    return saveData(processedData);
  })
  .then(() => {
    console.log('Saved successfully');
  })
  .catch(error => {
    console.error('Pipeline failed:', error);
  });
```

**Giải thích:**
- Mỗi `.then()` transform data và pass cho step tiếp theo
- Error ở bất kỳ step nào đều được catch ở cuối
- Pattern này tạo data processing pipeline

### 2. Promise Combinators

#### Promise.all() - Chạy Parallel, Đợi Tất Cả

```typescript
// Example 1: Promise.all() cơ bản
async function fetchMultipleUsers(userIds: string[]): Promise<User[]> {
  // Tạo array of promises
  const promises = userIds.map(id => getUser(id));

  // Đợi tất cả promises resolve
  const users = await Promise.all(promises);

  return users;
}

// Usage
const users = await fetchMultipleUsers(['1', '2', '3']);
// Nhanh hơn nhiều so với fetch tuần tự!
```

**Đặc điểm Promise.all():**
- Chạy tất cả promises **parallel** (đồng thời)
- Return array kết quả theo **thứ tự** input
- **Reject ngay lập tức** nếu bất kỳ promise nào fail
- Use case: Fetch multiple independent resources

```typescript
// Example 2: Promise.all() với mixed operations
async function loadDashboard(userId: string) {
  try {
    const [user, orders, notifications, settings] = await Promise.all([
      getUser(userId),
      getOrders(userId),
      getNotifications(userId),
      getSettings(userId),
    ]);

    return {
      user,
      orders,
      notifications,
      settings,
    };
  } catch (error) {
    // Nếu BẤT KỲ operation nào fail, vào đây
    console.error('Failed to load dashboard:', error);
    throw error;
  }
}
```

#### Promise.allSettled() - Đợi Tất Cả, Không Reject

```typescript
// Example 3: Promise.allSettled()
async function fetchUsersWithFallback(userIds: string[]): Promise<User[]> {
  const promises = userIds.map(id => getUser(id));

  // Đợi tất cả, không reject nếu có lỗi
  const results = await Promise.allSettled(promises);

  // Filter ra những kết quả thành công
  const users = results
    .filter((result): result is PromiseFulfilledResult<User> =>
      result.status === 'fulfilled'
    )
    .map(result => result.value);

  // Log những lỗi
  results
    .filter((result): result is PromiseRejectedResult =>
      result.status === 'rejected'
    )
    .forEach(result => {
      console.error('Failed to fetch user:', result.reason);
    });

  return users;
}
```

**Đặc điểm Promise.allSettled():**
- Đợi **tất cả** promises complete (fulfilled hoặc rejected)
- **Không bao giờ reject**
- Return array of objects: `{ status: 'fulfilled', value }` hoặc `{ status: 'rejected', reason }`
- Use case: Fetch data từ nhiều sources, muốn partial results

#### Promise.race() - Lấy Kết Quả Đầu Tiên

```typescript
// Example 4: Promise.race()
async function fetchWithTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number
): Promise<T> {
  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error('Timeout')), timeoutMs);
  });

  // Race giữa fetch và timeout
  return Promise.race([promise, timeoutPromise]);
}

// Usage
try {
  const user = await fetchWithTimeout(
    getUser('123'),
    5000 // 5 seconds timeout
  );
  console.log('User:', user);
} catch (error) {
  console.error('Fetch failed or timed out:', error);
}
```

**Đặc điểm Promise.race():**
- Return kết quả của promise **đầu tiên** settle (fulfilled hoặc rejected)
- Các promises khác vẫn chạy nhưng kết quả bị ignore
- Use case: Timeout, fallback to cache

#### Promise.any() - Lấy Success Đầu Tiên

```typescript
// Example 5: Promise.any()
async function fetchFromMultipleSources(userId: string): Promise<User> {
  // Try fetch từ nhiều sources
  const promises = [
    fetch(`https://api1.com/users/${userId}`).then(r => r.json()),
    fetch(`https://api2.com/users/${userId}`).then(r => r.json()),
    fetch(`https://api3.com/users/${userId}`).then(r => r.json()),
  ];

  try {
    // Lấy kết quả thành công đầu tiên
    const user = await Promise.any(promises);
    return user;
  } catch (error) {
    // Chỉ reject nếu TẤT CẢ promises đều fail
    console.error('All sources failed:', error);
    throw error;
  }
}
```

**Đặc điểm Promise.any():**
- Return promise **fulfilled đầu tiên**
- Ignore rejected promises
- Chỉ reject nếu **tất cả** promises reject (AggregateError)
- Use case: Redundant API calls, fallback chains

#### So Sánh Promise Combinators

| Combinator | Khi nào resolve? | Khi nào reject? | Use Case |
|------------|------------------|-----------------|----------|
| `Promise.all()` | Tất cả fulfilled | Bất kỳ promise nào reject | Fetch multiple required resources |
| `Promise.allSettled()` | Tất cả settled | Không bao giờ | Fetch với partial results OK |
| `Promise.race()` | Promise đầu tiên settled | Promise đầu tiên rejected | Timeout, fastest response |
| `Promise.any()` | Promise đầu tiên fulfilled | Tất cả rejected | Redundant sources, fallbacks |

### 3. Async/Await với Loops

#### Sequential Execution với for...of

```typescript
// Example 1: Sequential processing
async function processUsersSequentially(userIds: string[]): Promise<void> {
  for (const id of userIds) {
    // Đợi từng user xong mới process user tiếp theo
    const user = await getUser(id);
    await processUser(user);
    console.log(`Processed user ${id}`);
  }
  console.log('All users processed');
}

// Thời gian: n * (getUser + processUser)
```

**Khi nào dùng Sequential:**
- Operations phụ thuộc vào nhau
- Cần maintain order
- Rate limiting (tránh overwhelm server)
- Memory constraints (process từng item)

#### Parallel Execution với Promise.all()

```typescript
// Example 2: Parallel processing
async function processUsersParallel(userIds: string[]): Promise<void> {
  // Tạo array of promises
  const promises = userIds.map(async (id) => {
    const user = await getUser(id);
    await processUser(user);
    console.log(`Processed user ${id}`);
  });

  // Đợi tất cả xong
  await Promise.all(promises);
  console.log('All users processed');
}

// Thời gian: max(getUser + processUser) - nhanh hơn nhiều!
```

**Khi nào dùng Parallel:**
- Operations độc lập
- Muốn tối ưu performance
- Server có thể handle concurrent requests

#### Controlled Concurrency (Batch Processing)

```typescript
// Example 3: Process in batches
async function processInBatches<T>(
  items: T[],
  batchSize: number,
  processor: (item: T) => Promise<void>
): Promise<void> {
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);

    // Process batch parallel
    await Promise.all(batch.map(processor));

    console.log(`Processed batch ${i / batchSize + 1}`);
  }
}

// Usage
await processInBatches(
  userIds,
  10, // Process 10 users at a time
  async (id) => {
    const user = await getUser(id);
    await processUser(user);
  }
);
```

**Ưu điểm Batch Processing:**
- Balance giữa speed và resource usage
- Tránh overwhelm server
- Better error handling (fail một batch không ảnh hưởng batch khác)

#### Async map, filter, reduce

```typescript
// Example 4: Async map
async function asyncMap<T, U>(
  array: T[],
  asyncFn: (item: T) => Promise<U>
): Promise<U[]> {
  return Promise.all(array.map(asyncFn));
}

// Usage
const users = await asyncMap(userIds, getUser);

// Example 5: Async filter
async function asyncFilter<T>(
  array: T[],
  asyncPredicate: (item: T) => Promise<boolean>
): Promise<T[]> {
  const results = await Promise.all(array.map(asyncPredicate));
  return array.filter((_, index) => results[index]);
}

// Usage
const activeUsers = await asyncFilter(users, async (user) => {
  const status = await getUserStatus(user.id);
  return status === 'active';
});

// Example 6: Async reduce (sequential)
async function asyncReduce<T, U>(
  array: T[],
  asyncFn: (acc: U, item: T) => Promise<U>,
  initial: U
): Promise<U> {
  let accumulator = initial;
  for (const item of array) {
    accumulator = await asyncFn(accumulator, item);
  }
  return accumulator;
}

// Usage
const totalOrders = await asyncReduce(
  userIds,
  async (total, userId) => {
    const orders = await getOrders(userId);
    return total + orders.length;
  },
  0
);
```

### 4. Parallel vs Sequential Execution

#### Performance Comparison

```typescript
// Example 1: Sequential (slow)
async function fetchSequential() {
  console.time('Sequential');

  const user = await getUser('1');      // 1s
  const orders = await getOrders('1');  // 1s
  const reviews = await getReviews('1'); // 1s

  console.timeEnd('Sequential'); // ~3s

  return { user, orders, reviews };
}

// Example 2: Parallel (fast)
async function fetchParallel() {
  console.time('Parallel');

  const [user, orders, reviews] = await Promise.all([
    getUser('1'),      // 1s
    getOrders('1'),    // 1s (cùng lúc)
    getReviews('1'),   // 1s (cùng lúc)
  ]);

  console.timeEnd('Parallel'); // ~1s

  return { user, orders, reviews };
}
```

**Performance Gain:**
- Sequential: 3 seconds
- Parallel: 1 second (3x faster!)

#### When to Use Each

```typescript
// ✅ Sequential: Khi có dependencies
async function sequentialDependencies(userId: string) {
  // Phải có user trước
  const user = await getUser(userId);

  // Cần user.companyId
  const company = await getCompany(user.companyId);

  // Cần company.subscriptionId
  const subscription = await getSubscription(company.subscriptionId);

  return { user, company, subscription };
}

// ✅ Parallel: Khi độc lập
async function parallelIndependent(userId: string) {
  // Tất cả đều độc lập
  const [profile, settings, preferences, notifications] = await Promise.all([
    getUserProfile(userId),
    getUserSettings(userId),
    getUserPreferences(userId),
    getUserNotifications(userId),
  ]);

  return { profile, settings, preferences, notifications };
}
```

### 5. Error Handling Nâng Cao

#### Retry Logic

```typescript
// Example 1: Simple retry
async function fetchWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      console.log(`Attempt ${attempt} failed:`, error);

      if (attempt === maxRetries) {
        throw lastError;
      }

      // Wait before retry
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }

  throw lastError!;
}

// Usage
const user = await fetchWithRetry(() => getUser('123'), 3);
```

#### Exponential Backoff

```typescript
// Example 2: Retry với exponential backoff
async function fetchWithExponentialBackoff<T>(
  fn: () => Promise<T>,
  maxRetries: number = 5,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries - 1) {
        throw error;
      }

      // Exponential backoff: 1s, 2s, 4s, 8s, 16s
      const delay = baseDelay * Math.pow(2, attempt);
      console.log(`Retry after ${delay}ms`);

      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw new Error('Max retries exceeded');
}
```

**Giải thích Exponential Backoff:**
- Delay tăng theo cấp số nhân: 1s → 2s → 4s → 8s
- Giảm load lên server khi có vấn đề
- Tăng cơ hội thành công khi server recover

#### Partial Error Handling

```typescript
// Example 3: Handle partial failures
interface FetchResult<T> {
  success: T[];
  failed: Array<{ id: string; error: Error }>;
}

async function fetchMultipleWithErrors<T>(
  ids: string[],
  fetcher: (id: string) => Promise<T>
): Promise<FetchResult<T>> {
  const results = await Promise.allSettled(
    ids.map(id => fetcher(id).then(data => ({ id, data })))
  );

  const success: T[] = [];
  const failed: Array<{ id: string; error: Error }> = [];

  results.forEach((result, index) => {
    if (result.status === 'fulfilled') {
      success.push(result.value.data);
    } else {
      failed.push({
        id: ids[index],
        error: result.reason,
      });
    }
  });

  return { success, failed };
}

// Usage
const { success, failed } = await fetchMultipleWithErrors(
  ['1', '2', '3'],
  getUser
);

console.log(`Fetched ${success.length} users`);
console.log(`Failed ${failed.length} users:`, failed);
```

#### Error Recovery Strategies

```typescript
// Example 4: Fallback chain
async function getUserWithFallback(userId: string): Promise<User> {
  // Try 1: Fetch from API
  try {
    return await fetchUserFromAPI(userId);
  } catch (apiError) {
    console.warn('API failed, trying cache:', apiError);

    // Try 2: Get from cache
    try {
      const cached = await getUserFromCache(userId);
      if (cached) return cached;
    } catch (cacheError) {
      console.warn('Cache failed:', cacheError);
    }

    // Try 3: Get default user
    console.warn('Using default user');
    return getDefaultUser();
  }
}
```

### 6. Timeout và Cancellation

#### Timeout Pattern

```typescript
// Example 1: Generic timeout wrapper
function withTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number,
  timeoutError: Error = new Error('Operation timed out')
): Promise<T> {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => reject(timeoutError), timeoutMs);

    promise
      .then(value => {
        clearTimeout(timer);
        resolve(value);
      })
      .catch(error => {
        clearTimeout(timer);
        reject(error);
      });
  });
}

// Usage
try {
  const user = await withTimeout(
    getUser('123'),
    5000,
    new Error('User fetch timed out')
  );
} catch (error) {
  console.error(error);
}
```

#### AbortController (Modern Approach)

```typescript
// Example 2: Cancellable fetch với AbortController
async function fetchWithCancel(
  url: string,
  timeoutMs: number
): Promise<any> {
  const controller = new AbortController();
  const { signal } = controller;

  // Set timeout
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, { signal });
    clearTimeout(timeoutId);
    return await response.json();
  } catch (error) {
    clearTimeout(timeoutId);

    if (error instanceof Error && error.name === 'AbortError') {
      throw new Error('Request was cancelled');
    }
    throw error;
  }
}

// Usage
try {
  const data = await fetchWithCancel('/api/data', 5000);
} catch (error) {
  console.error('Fetch failed:', error);
}
```

#### Manual Cancellation

```typescript
// Example 3: Cancellable operation
class CancellableOperation<T> {
  private cancelled = false;

  cancel(): void {
    this.cancelled = true;
  }

  async execute(fn: () => Promise<T>): Promise<T> {
    if (this.cancelled) {
      throw new Error('Operation cancelled');
    }

    return await fn();
  }

  isCancelled(): boolean {
    return this.cancelled;
  }
}

// Usage
const operation = new CancellableOperation<User>();

// Start operation
const userPromise = operation.execute(async () => {
  const user = await getUser('123');

  // Check cancellation between steps
  if (operation.isCancelled()) {
    throw new Error('Cancelled');
  }

  const orders = await getOrders(user.id);
  return { ...user, orders };
});

// Cancel after 2 seconds
setTimeout(() => operation.cancel(), 2000);

try {
  const user = await userPromise;
} catch (error) {
  console.error('Operation cancelled or failed:', error);
}
```

#### Race with Timeout

```typescript
// Example 4: Multiple operations với timeout
async function raceWithTimeout<T>(
  operations: Array<() => Promise<T>>,
  timeoutMs: number
): Promise<T> {
  const timeout = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error('All operations timed out')), timeoutMs);
  });

  const promises = operations.map(op => op());

  return Promise.race([...promises, timeout]);
}

// Usage
const user = await raceWithTimeout(
  [
    () => fetchFromAPI1('123'),
    () => fetchFromAPI2('123'),
    () => fetchFromCache('123'),
  ],
  3000
);
```

---

## Best Practices - Junior & Middle Level

### 1. Luôn Handle Errors

```typescript
// ❌ BAD
async function bad() {
  const data = await fetch('/api/data');
  return data.json();
}

// ✅ GOOD
async function good() {
  try {
    const response = await fetch('/api/data');
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error;
  }
}
```

### 2. Tránh Floating Promises

```typescript
// ❌ BAD
async function bad() {
  saveToDatabase(data); // Không await!
  return 'Done';
}

// ✅ GOOD
async function good() {
  await saveToDatabase(data);
  return 'Done';
}
```

### 3. Sử Dụng Promise.all() Cho Parallel Operations

```typescript
// ❌ BAD - Sequential
async function bad(ids: string[]) {
  const results = [];
  for (const id of ids) {
    results.push(await fetchData(id)); // Chậm!
  }
  return results;
}

// ✅ GOOD - Parallel
async function good(ids: string[]) {
  return Promise.all(ids.map(id => fetchData(id)));
}
```

### 4. Proper TypeScript Typing

```typescript
// ❌ BAD
async function bad(id: any): Promise<any> {
  return await fetch(`/api/${id}`);
}

// ✅ GOOD
async function good(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new Error(`Failed to fetch user ${id}`);
  }
  return await response.json();
}
```

### 5. Cleanup trong Finally

```typescript
// ✅ GOOD
async function processWithCleanup() {
  const connection = await openConnection();

  try {
    await processData(connection);
  } catch (error) {
    console.error('Processing failed:', error);
    throw error;
  } finally {
    // Luôn cleanup
    await connection.close();
  }
}
```

---

## Tài Liệu Tham Khảo

### Official Documentation
- [TypeScript Handbook - Async/Await](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-1-7.html)
- [MDN - Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [MDN - Async Functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)

### Advanced Topics
- Xem `Advanced-Async-Patterns.md` cho Senior level patterns
- Xem `Principal-Async-Patterns.md` cho Enterprise patterns

---

**Tài liệu này bao gồm Junior và Middle levels. Để học các patterns nâng cao hơn, hãy tiếp tục với các files khác trong thư mục này.**

