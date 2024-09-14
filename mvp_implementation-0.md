# Updated Step-by-Step Coding Solution for Prox MVP

This updated guide incorporates React Query (RTK Query) for state management, shadcn components for UI, and PostgreSQL as the database.

## **Project Overview**

1. **Frontend:**
   - **Framework:** React with TypeScript
   - **UI Library:** shadcn
   - **State Management:** RTK Query
   - **Routing:** React Router
   - **Styling:** Tailwind CSS (since shadcn is built on top of Tailwind)

2. **Backend:**
   - **Server:** Node.js with Express
   - **Database:** PostgreSQL
   - **Authentication:** JWT (JSON Web Tokens)
   - **AI Integration:** OpenAI API for profile generation and natural language search
   - **Hosting:** Vercel or Heroku for frontend; AWS, DigitalOcean, or similar for backend

3. **Other Tools:**
   - **Version Control:** GitHub
   - **API Documentation:** Swagger or Postman
   - **Environment Management:** dotenv

---

## **Step 1: Set Up the Development Environment**

### **1.1. Initialize the Project**

```bash
# Create a new React app with TypeScript
npx create-react-app prox-mvp --template typescript
cd prox-mvp

# Initialize a Git repository
git init
```

### **1.2. Install Dependencies**

```bash
# Install shadcn and Tailwind CSS dependencies
npx shadcn-ui@latest init

# Install additional dependencies
npm install axios react-router-dom @types/react-router-dom @reduxjs/toolkit react-redux
```

### **1.3. Set Up PostgreSQL**

1. Install PostgreSQL on your system.
2. Create a new database for your project:

```sql
CREATE DATABASE prox_mvp;
```

3. Install the PostgreSQL client for Node.js:

```bash
npm install pg
```

---

## **Step 2: Set Up the Backend**

### **2.1. Initialize the Backend**

```bash
# In the root directory
mkdir backend
cd backend
npm init -y

# Install backend dependencies
npm install express cors dotenv jsonwebtoken bcryptjs pg
npm install -D typescript @types/express @types/node @types/pg nodemon ts-node
```

### **2.2. Configure TypeScript**

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES6",
    "module": "CommonJS",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true
  }
}
```

### **2.3. Set Up Express Server with PostgreSQL**

```typescript
// backend/src/index.ts
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';
import { Pool } from 'pg';
import authRoutes from './routes/auth';
import profileRoutes from './routes/profile';
import searchRoutes from './routes/search';

dotenv.config();

const app = express();

app.use(cors());
app.use(express.json());

// PostgreSQL connection
export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

app.use('/api/auth', authRoutes);
app.use('/api/profile', profileRoutes);
app.use('/api/search', searchRoutes);

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### **2.4. Create Authentication Routes**

```typescript
// backend/src/routes/auth.ts
import express from 'express';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { pool } from '../index';

const router = express.Router();

// Register
router.post('/register', async (req, res) => {
  const { email, password } = req.body;
  try {
    const existingUser = await pool.query('SELECT * FROM users WHERE email = $1', [email]);
    if (existingUser.rows.length)
      return res.status(400).json({ message: 'User already exists' });

    const hashedPassword = await bcrypt.hash(password, 12);
    const newUser = await pool.query(
      'INSERT INTO users (email, password) VALUES ($1, $2) RETURNING id, email',
      [email, hashedPassword]
    );

    const token = jwt.sign({ id: newUser.rows[0].id }, process.env.JWT_SECRET!, {
      expiresIn: '1h',
    });

    res.status(201).json({ token, userId: newUser.rows[0].id });
  } catch (error) {
    res.status(500).json({ message: 'Something went wrong' });
  }
});

// Login
router.post('/login', async (req, res) => {
  const { email, password } = req.body;
  try {
    const user = await pool.query('SELECT * FROM users WHERE email = $1', [email]);
    if (!user.rows.length)
      return res.status(400).json({ message: 'Invalid credentials' });

    const isMatch = await bcrypt.compare(password, user.rows[0].password);
    if (!isMatch)
      return res.status(400).json({ message: 'Invalid credentials' });

    const token = jwt.sign({ id: user.rows[0].id }, process.env.JWT_SECRET!, {
      expiresIn: '1h',
    });

    res.status(200).json({ token, userId: user.rows[0].id });
  } catch (error) {
    res.status(500).json({ message: 'Something went wrong' });
  }
});

export default router;
```

---

## **Step 3: Implement RTK Query on the Frontend**

### **3.1. Set Up RTK Query**

Create a new file `src/api/api.ts`:

```typescript
// src/api/api.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: 'http://localhost:5000/api' }),
  endpoints: (builder) => ({
    login: builder.mutation({
      query: (credentials) => ({
        url: 'auth/login',
        method: 'POST',
        body: credentials,
      }),
    }),
    register: builder.mutation({
      query: (userData) => ({
        url: 'auth/register',
        method: 'POST',
        body: userData,
      }),
    }),
    getProfile: builder.query({
      query: (userId) => `profile/${userId}`,
    }),
    updateProfile: builder.mutation({
      query: ({ userId, ...patch }) => ({
        url: `profile/${userId}`,
        method: 'PATCH',
        body: patch,
      }),
    }),
    search: builder.mutation({
      query: (searchQuery) => ({
        url: 'search',
        method: 'POST',
        body: { query: searchQuery },
      }),
    }),
  }),
});

export const {
  useLoginMutation,
  useRegisterMutation,
  useGetProfileQuery,
  useUpdateProfileMutation,
  useSearchMutation,
} = api;
```

### **3.2. Set Up Redux Store**

Create a new file `src/store/store.ts`:

```typescript
// src/store/store.ts
import { configureStore } from '@reduxjs/toolkit';
import { api } from '../api/api';

export const store = configureStore({
  reducer: {
    [api.reducerPath]: api.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(api.middleware),
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### **3.3. Wrap App with Redux Provider**

Update `src/index.tsx`:

```typescript
// src/index.tsx
import React from 'react';
import ReactDOM from 'react-dom';
import { Provider } from 'react-redux';
import { store } from './store/store';
import './index.css';
import App from './App';

ReactDOM.render(
  <React.StrictMode>
    <Provider store={store}>
      <App />
    </Provider>
  </React.StrictMode>,
  document.getElementById('root')
);
```

---

## **Step 4: Update Components to Use RTK Query and shadcn**

### **4.1. Update Login Component**

```typescript
// src/pages/Login.tsx
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useLoginMutation } from '../api/api';
import { Button, Input, Form } from '@/components/ui/form';

const Login: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const navigate = useNavigate();

  const [login, { isLoading }] = useLoginMutation();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      const result = await login({ email, password }).unwrap();
      localStorage.setItem('token', result.token);
      localStorage.setItem('userId', result.userId);
      navigate('/');
    } catch (error) {
      console.error('Login failed:', error);
    }
  };

  return (
    <div className="flex items-center justify-center h-screen">
      <Form onSubmit={handleSubmit} className="w-1/3">
        <h2 className="text-2xl mb-4">Login</h2>
        <Input
          type="email"
          placeholder="Email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
          className="mb-4"
        />
        <Input
          type="password"
          placeholder="Password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
          className="mb-4"
        />
        <Button type="submit" disabled={isLoading}>
          {isLoading ? 'Logging in...' : 'Login'}
        </Button>
      </Form>
    </div>
  );
};

export default Login;
```

### **4.2. Update Dashboard Component**

```typescript
// src/pages/Dashboard.tsx
import React from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { useGetProfileQuery } from '../api/api';
import { Button, Card } from '@/components/ui/card';

const Dashboard: React.FC = () => {
  const navigate = useNavigate();
  const userId = localStorage.getItem('userId');
  const { data: profile, isLoading, error } = useGetProfileQuery(userId);

  const handleLogout = () => {
    localStorage.removeItem('token');
    localStorage.removeItem('userId');
    navigate('/login');
  };

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading profile</div>;

  return (
    <div className="p-8">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-3xl">Dashboard</h1>
        <div>
          <Link to="/search">
            <Button className="mr-4">Search</Button>
          </Link>
          <Button onClick={handleLogout}>Logout</Button>
        </div>
      </div>
      <Card className="mb-6">
        <h2 className="text-2xl mb-4">Your Profile</h2>
        {profile && (
          <div>
            <p><strong>Name:</strong> {profile.name}</p>
            <p><strong>Email:</strong> {profile.email}</p>
            {/* Display other profile fields */}
          </div>
        )}
      </Card>
      {/* Add more dashboard content here */}
    </div>
  );
};

export default Dashboard;
```

### **4.3. Update Search Component**

```typescript
// src/pages/Search.tsx
import React, { useState } from 'react';
import { useSearchMutation } from '../api/api';
import { Button, Input, Card } from '@/components/ui/card';

const Search: React.FC = () => {
  const [query, setQuery] = useState('');
  const [search, { data: results, isLoading }] = useSearchMutation();

  const handleSearch = async () => {
    if (!query) return;
    await search(query);
  };

  return (
    <div className="p-8">
      <Card className="mb-6">
        <h2 className="text-2xl mb-4">Search Professionals</h2>
        <Input
          type="text"
          placeholder="Enter your search query"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          className="mb-4"
        />
        <Button onClick={handleSearch} disabled={isLoading}>
          {isLoading ? 'Searching...' : 'Search'}
        </Button>
      </Card>
      <div>
        {results && results.map((result: any, index: number) => (
          <Card key={index} className="mb-4">
            <p>{result.content}</p>
          </Card>
        ))}
      </div>
    </div>
  );
};

export default Search;
```

---

## **Step 5: Update Backend to Use PostgreSQL**

### **5.1. Create Database Schema**

Run the following SQL commands to create the necessary tables:

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL,
  name VARCHAR(255),
  comprehensive_profile TEXT
);

CREATE TABLE links (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  url VARCHAR(255) NOT NULL
);
```

### **5.2. Update Profile Routes**

```typescript
// backend/src/routes/profile.ts
import express from 'express';
import jwt from 'jsonwebtoken';
import { pool } from '../index';

const router = express.Router();

// Middleware to authenticate the user
const authenticate = (req: any, res: any, next: any) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token)
    return res.status(401).json({ message: 'No token provided' });

  try {
    const decoded: any = jwt.verify(token, process.env.JWT_SECRET!);
    req.userId = decoded.id;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

// Get user profile
router.get('/:userId', authenticate, async (req, res) => {
  const { userId } = req.params;
  try {
    const result = await pool.query('SELECT id, email, name, comprehensive_profile FROM users WHERE id = $1', [userId]);
    if (!result.rows.length)
      return res.status(404).json({ message: 'User not found' });
    res.status(200).json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ message: 'Something went wrong' });
  }
});

// Add a link to the user's profile
router.post('/:userId/links', authenticate, async (req, res) => {
  const { userId } = req.params;
  const { link } = req.body;
  try {
    // Simple validation for link
    if (!/^https?:\/\/.+/.test(link))
      return res.status(400).json({ message: 'Invalid URL' });

    await pool.query('INSERT INTO links (user_id, url) VALUES ($1, $2)', [userId, link]);

    // TODO: Trigger AI profile aggregation here

    res.status(200).json({ message: 'Link added successfully' });
  } catch (error) {
    res.status(500).json({ message: 'Something went wrong' });
  }
});

export default router;
```

### **5.3. Update Search Routes**

```typescript
// backend/src/routes/search.ts
import express from 'express';
import jwt from 'jsonwebtoken';
import { pool } from '../index';
import axios from 'axios';

const router = express.Router();

// Middleware to authenticate the user
const authenticate = (req: any, res: any, next: any) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token)
    return res.status(401).json({ message: 'No token provided' });

  try {
    const decoded: any = jwt.verify(token, process.env.JWT_SECRET!);
    req.userId = decoded.id;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

// Search using Natural Language Query
router.post('/', authenticate, async (req, res) => {
  const { query } = req.body;
  try {
    // Use OpenAI to interpret the query and fetch relevant profiles
    const aiResponse = await axios.post(
      'https://api.openai.com/v1/engines/text-davinci-003/completions',
      {
        prompt: `Search the following professional profiles and return relevant results based on this query:\n\nQuery: ${query}\n\nProfiles:\n${await getAllProfilesAsText()}\n\nResults:`,
        max_tokens: 500,
      },
      {
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
        },
      }
    );

    res.status(200).json({ results: aiResponse.data.choices[0].text });
  } catch (error) {
    console.error('Search failed:', error);
    res.status(500).json({ message: 'Something went wrong' });
  }
});

// Helper function to fetch all profiles as text
const getAllProfilesAsText = async () => {
  const result = await pool.query('SELECT comprehensive_profile FROM users');
  return result.rows.map((row) => row.comprehensive_profile).join('\n');
};

export default router;
```

---

## **Step 6: Implement Unique Digital Identifiers (QR Codes)**

### **6.1. Install QR Code Library**

```bash
npm install qrcode.react
```

### **6.2. Update Dashboard Component to Include QR Code**

```typescript
// src/pages/Dashboard.tsx
import React from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { useGetProfileQuery } from '../api/api';
import { Button, Card } from '@/components/ui/card';
import QRCode from 'qrcode.react';

const Dashboard: React.FC = () => {
  const navigate = useNavigate();
  const userId = localStorage.getItem('userId');
  const { data: profile, isLoading, error } = useGetProfileQuery(userId);

  const handleLogout = () => {
    localStorage.removeItem('token');
    localStorage.removeItem('userId');
    navigate('/login');
  };

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading profile</div>;

  return (
    <div className="p-8">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-3xl">Dashboard</h1>
        <div>
          <Link to="/search">
            <Button className="mr-4">Search</Button>
          </Link>
          <Button onClick={handleLogout}>Logout</Button>
        </div>
      </div>
      <Card className="mb-6">
        <h2 className="text-2xl mb-4">Your Profile</h2>
        {profile && (
          <div>
            <p><strong>Name:</strong> {profile.name}</p>
            <p><strong>Email:</strong> {profile.email}</p>
            <div className="mt-4">
              <h3 className="text-xl mb-2">Your QR Code</h3>
              <QRCode value={`https://prox.com/profile/${userId}`} />
            </div>
          </div>
        )}
      </Card>
    </div>
  );
};

export default Dashboard;
```

### **6.3. Create Public Profile Page**

```typescript
// src/pages/PublicProfile.tsx
import React from 'react';
import { useParams } from 'react-router-dom';
import { useGetProfileQuery } from '../api/api';
import { Card } from '@/components/ui/card';

const PublicProfile: React.FC = () => {
  const { userId } = useParams<{ userId: string }>();
  const { data: profile, isLoading, error } = useGetProfileQuery(userId);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading profile</div>;

  return (
    <div className="p-8">
      <Card>
        {profile && (
          <div>
            <h2 className="text-2xl mb-4">{profile.name}</h2>
            <p><strong>Email:</strong> {profile.email}</p>
            <div className="mt-4">
              <h3 className="text-xl mb-2">Profile Overview</h3>
              <p>{profile.comprehensive_profile}</p>
            </div>
          </div>
        )}
      </Card>
    </div>
  );
};

export default PublicProfile;
```

---

## **Step 7: Implement Scheduling and Calendar Integration**

### **7.1. Set Up Google API Credentials**

Follow the steps outlined in the previous implementation to set up Google API credentials.

### **7.2. Create Scheduling Routes on the Backend**

```typescript
// backend/src/routes/schedule.ts
import express from 'express';
import jwt from 'jsonwebtoken';
import { google } from 'googleapis';
import { pool } from '../index';

const router = express.Router();

// Middleware to authenticate the user
const authenticate = (req: any, res: any, next: any) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token)
    return res.status(401).json({ message: 'No token provided' });

  try {
    const decoded: any = jwt.verify(token, process.env.JWT_SECRET!);
    req.userId = decoded.id;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

// Create a new event
router.post('/event', authenticate, async (req, res) => {
  const { summary, description, start, end, attendees } = req.body;
  try {
    // Initialize OAuth2 client
    const oauth2Client = new google.auth.OAuth2(
      process.env.GOOGLE_CLIENT_ID,
      process.env.GOOGLE_CLIENT_SECRET,
      process.env.GOOGLE_REDIRECT_URI
    );

    // Fetch user's tokens from the database
    const result = await pool.query('SELECT access_token, refresh_token FROM users WHERE id = $1', [req.userId]);
    const { access_token, refresh_token } = result.rows[0];

    oauth2Client.setCredentials({
      access_token,
      refresh_token,
      scope: 'https://www.googleapis.com/auth/calendar',
      token_type: 'Bearer',
      expiry_date: true,
    });

    const calendar = google.calendar({ version: 'v3', auth: oauth2Client });

    const event = {
      summary,
      description,
      start: { dateTime: start },
      end: { dateTime: end },
      attendees: attendees.map((email: string) => ({ email })),
    };

    const response = await calendar.events.insert({
      calendarId: 'primary',
      requestBody: event,
    });

    res.status(200).json({ event: response.data });
  } catch (error) {
    console.error('Failed to create event:', error);
    res.status(500).json({ message: 'Something went wrong' });
  }
});

export default router;
```

### **7.3. Create Scheduler Component on the Frontend**

```typescript
// src/pages/Scheduler.tsx
import React, { useState } from 'react';
import { useCreateEventMutation } from '../api/api';
import { Button, Input, Textarea, Form } from '@/components/ui/form';

const Scheduler: React.FC = () => {
  const [summary, setSummary] = useState('');
  const [description, setDescription] = useState('');
  const [start, setStart] = useState('');
  const [end, setEnd] = useState('');
  const [attendees, setAttendees] = useState('');

  const [createEvent, { isLoading }] = useCreateEventMutation();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await createEvent({
        summary,
        description,
        start,
        end,
        attendees: attendees.split(',').map(email => email.trim()),
      }).unwrap();
      // Clear form
      setSummary('');
      setDescription('');
      setStart('');
      setEnd('');
      setAttendees('');
      // Optionally notify user
    } catch (error) {
      console.error('Failed to create event:', error);
      // Handle error (e.g., display message)
    }
  };

  return (
    <div className="p-8">
      <Form onSubmit={handleSubmit} className="w-1/2 mx-auto">
        <h2 className="text-2xl mb-4">Schedule an Event</h2>
        <Input
          type="text"
          placeholder="Event Summary"
          value={summary}
          onChange={(e) => setSummary(e.target.value)}
          required
          className="mb-4"
        />
        <Textarea
          placeholder="Event Description"
          value={description}
          onChange={(e) => setDescription(e.target.value)}
          className="mb-4"
        />
        <Input
          type="datetime-local"
          placeholder="Start Time"
          value={start}
          onChange={(e) => setStart(e.target.value)}
          required
          className="mb-4"
        />
        <Input
          type="datetime-local"
          placeholder="End Time"
          value={end}
          onChange={(e) => setEnd(e.target.value)}
          required
          className="mb-4"
        />
        <Input
          type="text"
          placeholder="Attendees (comma-separated emails)"
          value={attendees}
          onChange={(e) => setAttendees(e.target.value)}
          className="mb-4"
        />
        <Button type="submit" disabled={isLoading}>
          {isLoading ? 'Scheduling...' : 'Schedule Event'}
        </Button>
      </Form>
    </div>
  );
};

export default Scheduler;
```

### **7.4. Update API Slice to Include Event Creation**

```typescript
// src/api/api.ts
// ... existing code

export const api = createApi({
  // ... existing configuration
  endpoints: (builder) => ({
    // ... existing endpoints
    createEvent: builder.mutation({
      query: (eventData) => ({
        url: 'schedule/event',
        method: 'POST',
        body: eventData,
      }),
    }),
  }),
});

export const {
  // ... existing exports
  useCreateEventMutation,
} = api;
```

### **7.5. Update App Component to Include Scheduler Route**

```typescript
// src/App.tsx
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Login from './pages/Login';
import Register from './pages/Register';
import Dashboard from './pages/Dashboard';
import Search from './pages/Search';
import PublicProfile from './pages/PublicProfile';
import Scheduler from './pages/Scheduler';
import PrivateRoute from './components/PrivateRoute';

const App: React.FC = () => {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<PrivateRoute><Dashboard /></PrivateRoute>} />
        <Route path="/search" element={<PrivateRoute><Search /></PrivateRoute>} />
        <Route path="/scheduler" element={<PrivateRoute><Scheduler /></PrivateRoute>} />
        <Route path="/profile/:userId" element={<PublicProfile />} />
        <Route path="/login" element={<Login />} />
        <Route path="/register" element={<Register />} />
      </Routes>
    </Router>
  );
};

export default App;
```

This updated implementation incorporates RTK Query for state management, shadcn components for UI, and PostgreSQL as the database. It maintains the core functionality of the Prox MVP while leveraging these technologies for improved performance and developer experience.

Remember to update your environment variables, database connection strings, and API endpoints as needed. Also, ensure that you have the necessary PostgreSQL database set up and running before starting your backend server.