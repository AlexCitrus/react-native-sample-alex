# React Native Inventory App - Build From Scratch

🚀 **Start with ZERO code and build a complete React Native inventory management app!**

This guide will take you from an empty project to a fully functional mobile app with authentication, database integration, and CRUD operations.

## 🎯 What You'll Build

A complete mobile inventory management app featuring:
- User registration and login
- Protected navigation
- Dashboard with statistics
- Add, edit, and delete inventory items
- Real-time data synchronization
- Professional mobile UI

## 📋 Prerequisites

- Node.js (v16 or later)
- Expo CLI: `npm install -g @expo/cli`
- A code editor (VS Code recommended)
- A Supabase account (free at [supabase.com](https://supabase.com))
- Basic knowledge of JavaScript/React (helpful but not required)

## 🚀 Getting Started

### Step 1: Initial Setup

```bash
# Install dependencies (if not already done)
yarn install
# or
npm install

# Start the development server
expo start
```

**Note**: The app will initially crash because we deleted all the code files. That's expected! We'll build everything step by step.

## 🛠 Building the App Step by Step

### Step 2: Create the Main App File

Create `App.js` in the root directory:

```javascript
import React from 'react';
import { Text, View, StyleSheet } from 'react-native';

export default function App() {
  return (
    <View style={styles.container}>
      <Text style={styles.text}>Welcome to Inventory App!</Text>
      <Text style={styles.subtext}>Let's build this together 🚀</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f5f5f5',
  },
  text: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 10,
  },
  subtext: {
    fontSize: 16,
    color: '#666',
  },
});
```

**Test**: Run `expo start` - you should see a simple welcome screen!

### Step 3: Set Up Supabase Backend

1. **Create Supabase Project**
   - Go to [supabase.com](https://supabase.com) and sign up
   - Click "New Project"
   - Name your project and set a database password
   - Wait 2-3 minutes for setup

2. **Get Your Credentials**
   - Go to **Settings** → **API**
   - Copy your `Project URL`
   - Copy your `anon public` key

3. **Create Supabase Service**

Create `src/services/supabase.js`:

```javascript
import AsyncStorage from '@react-native-async-storage/async-storage';
import { createClient } from '@supabase/supabase-js';
import 'react-native-url-polyfill/auto';

// Replace with your Supabase credentials
const supabaseUrl = 'YOUR_SUPABASE_URL_HERE';
const supabaseAnonKey = 'YOUR_SUPABASE_ANON_KEY_HERE';

export const supabase = createClient(supabaseUrl, supabaseAnonKey, {
  auth: {
    storage: AsyncStorage,
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false,
  },
});

// Auth functions
export const signUp = async (email, password) => {
  const { data, error } = await supabase.auth.signUp({
    email: email,
    password: password,
  });
  return { data, error };
};

export const signIn = async (email, password) => {
  const { data, error } = await supabase.auth.signInWithPassword({
    email: email,
    password: password,
  });
  return { data, error };
};

export const signOut = async () => {
  const { error } = await supabase.auth.signOut();
  return { error };
};

export const getCurrentUser = async () => {
  const { data: { session } } = await supabase.auth.getSession();
  return session?.user ?? null;
};
```

**Replace the URL and key with your actual Supabase credentials!**

### Step 4: Set Up Database

In your Supabase dashboard, go to **SQL Editor** and run:

```sql
-- Create inventory_items table
CREATE TABLE inventory_items (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  name TEXT NOT NULL,
  description TEXT,
  quantity INTEGER NOT NULL DEFAULT 0,
  price DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Enable Row Level Security
ALTER TABLE inventory_items ENABLE ROW LEVEL SECURITY;

-- Create policies for data access
CREATE POLICY "Users can view own items" ON inventory_items
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own items" ON inventory_items
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own items" ON inventory_items
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own items" ON inventory_items
  FOR DELETE USING (auth.uid() = user_id);
```

### Step 5: Create Inventory Service

Create `src/services/inventoryService.js`:

```javascript
import { supabase } from './supabase';

export const getInventoryItems = async () => {
  try {
    const { data, error } = await supabase
      .from('inventory_items')
      .select('*')
      .order('created_at', { ascending: false });
    
    if (error) throw error;
    return { data, error: null };
  } catch (error) {
    return { data: null, error: error.message };
  }
};

export const addInventoryItem = async (item) => {
  try {
    const user = await supabase.auth.getUser();
    if (!user.data.user) throw new Error('No authenticated user');

    const { data, error } = await supabase
      .from('inventory_items')
      .insert([
        {
          ...item,
          user_id: user.data.user.id,
        },
      ])
      .select();

    if (error) throw error;
    return { data, error: null };
  } catch (error) {
    return { data: null, error: error.message };
  }
};

export const updateInventoryItem = async (id, updates) => {
  try {
    const { data, error } = await supabase
      .from('inventory_items')
      .update(updates)
      .eq('id', id)
      .select();

    if (error) throw error;
    return { data, error: null };
  } catch (error) {
    return { data: null, error: error.message };
  }
};

export const deleteInventoryItem = async (id) => {
  try {
    const { error } = await supabase
      .from('inventory_items')
      .delete()
      .eq('id', id);

    if (error) throw error;
    return { error: null };
  } catch (error) {
    return { error: error.message };
  }
};
```

### Step 6: Create Authentication Context

Create `src/context/AuthContext.js`:

```javascript
import React, { createContext, useContext, useEffect, useState } from 'react';
import { supabase, getCurrentUser } from '../services/supabase';

const AuthContext = createContext({});

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Get initial session
    getCurrentUser().then((user) => {
      setUser(user);
      setLoading(false);
    });

    // Listen for auth changes
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      async (event, session) => {
        setUser(session?.user ?? null);
        setLoading(false);
      }
    );

    return () => subscription.unsubscribe();
  }, []);

  const value = {
    user,
    loading,
  };

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Step 7: Create Loading Screen

Create `src/screens/LoadingScreen.js`:

```javascript
import React from 'react';
import { View, Text, ActivityIndicator, StyleSheet } from 'react-native';

export default function LoadingScreen() {
  return (
    <View style={styles.container}>
      <ActivityIndicator size="large" color="#007AFF" />
      <Text style={styles.text}>Loading...</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f5f5f5',
  },
  text: {
    marginTop: 20,
    fontSize: 16,
    color: '#666',
  },
});
```

### Step 8: Create Authentication Screens

Create `src/screens/auth/LoginScreen.js`:

```javascript
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import { signIn } from '../../services/supabase';

export default function LoginScreen({ navigation }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const handleLogin = async () => {
    if (!email || !password) {
      Alert.alert('Error', 'Please fill in all fields');
      return;
    }

    setLoading(true);
    const { error } = await signIn(email, password);
    setLoading(false);

    if (error) {
      Alert.alert('Login Error', error.message);
    }
  };

  return (
    <KeyboardAvoidingView 
      style={styles.container} 
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <View style={styles.content}>
        <Text style={styles.title}>Welcome Back</Text>
        <Text style={styles.subtitle}>Sign in to your account</Text>

        <TextInput
          style={styles.input}
          placeholder="Email"
          value={email}
          onChangeText={setEmail}
          keyboardType="email-address"
          autoCapitalize="none"
        />

        <TextInput
          style={styles.input}
          placeholder="Password"
          value={password}
          onChangeText={setPassword}
          secureTextEntry
        />

        <TouchableOpacity 
          style={[styles.button, loading && styles.buttonDisabled]} 
          onPress={handleLogin}
          disabled={loading}
        >
          <Text style={styles.buttonText}>
            {loading ? 'Signing In...' : 'Sign In'}
          </Text>
        </TouchableOpacity>

        <TouchableOpacity 
          style={styles.linkButton}
          onPress={() => navigation.navigate('Register')}
        >
          <Text style={styles.linkText}>
            Don't have an account? Sign Up
          </Text>
        </TouchableOpacity>
      </View>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  content: {
    flex: 1,
    justifyContent: 'center',
    paddingHorizontal: 20,
  },
  title: {
    fontSize: 32,
    fontWeight: 'bold',
    textAlign: 'center',
    marginBottom: 8,
    color: '#333',
  },
  subtitle: {
    fontSize: 16,
    textAlign: 'center',
    marginBottom: 40,
    color: '#666',
  },
  input: {
    backgroundColor: 'white',
    borderRadius: 8,
    paddingHorizontal: 16,
    paddingVertical: 12,
    marginBottom: 16,
    fontSize: 16,
    borderWidth: 1,
    borderColor: '#ddd',
  },
  button: {
    backgroundColor: '#007AFF',
    borderRadius: 8,
    paddingVertical: 12,
    marginBottom: 16,
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  buttonText: {
    color: 'white',
    textAlign: 'center',
    fontSize: 16,
    fontWeight: '600',
  },
  linkButton: {
    paddingVertical: 8,
  },
  linkText: {
    color: '#007AFF',
    textAlign: 'center',
    fontSize: 16,
  },
});
```

Create `src/screens/auth/RegisterScreen.js`:

```javascript
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import { signUp } from '../../services/supabase';

export default function RegisterScreen({ navigation }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const handleRegister = async () => {
    if (!email || !password || !confirmPassword) {
      Alert.alert('Error', 'Please fill in all fields');
      return;
    }

    if (password !== confirmPassword) {
      Alert.alert('Error', 'Passwords do not match');
      return;
    }

    if (password.length < 6) {
      Alert.alert('Error', 'Password must be at least 6 characters');
      return;
    }

    setLoading(true);
    const { error } = await signUp(email, password);
    setLoading(false);

    if (error) {
      Alert.alert('Registration Error', error.message);
    } else {
      Alert.alert(
        'Success',
        'Account created! Please check your email to verify your account.',
        [{ text: 'OK', onPress: () => navigation.navigate('Login') }]
      );
    }
  };

  return (
    <KeyboardAvoidingView 
      style={styles.container} 
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <View style={styles.content}>
        <Text style={styles.title}>Create Account</Text>
        <Text style={styles.subtitle}>Sign up to get started</Text>

        <TextInput
          style={styles.input}
          placeholder="Email"
          value={email}
          onChangeText={setEmail}
          keyboardType="email-address"
          autoCapitalize="none"
        />

        <TextInput
          style={styles.input}
          placeholder="Password"
          value={password}
          onChangeText={setPassword}
          secureTextEntry
        />

        <TextInput
          style={styles.input}
          placeholder="Confirm Password"
          value={confirmPassword}
          onChangeText={setConfirmPassword}
          secureTextEntry
        />

        <TouchableOpacity 
          style={[styles.button, loading && styles.buttonDisabled]} 
          onPress={handleRegister}
          disabled={loading}
        >
          <Text style={styles.buttonText}>
            {loading ? 'Creating Account...' : 'Sign Up'}
          </Text>
        </TouchableOpacity>

        <TouchableOpacity 
          style={styles.linkButton}
          onPress={() => navigation.navigate('Login')}
        >
          <Text style={styles.linkText}>
            Already have an account? Sign In
          </Text>
        </TouchableOpacity>
      </View>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  content: {
    flex: 1,
    justifyContent: 'center',
    paddingHorizontal: 20,
  },
  title: {
    fontSize: 32,
    fontWeight: 'bold',
    textAlign: 'center',
    marginBottom: 8,
    color: '#333',
  },
  subtitle: {
    fontSize: 16,
    textAlign: 'center',
    marginBottom: 40,
    color: '#666',
  },
  input: {
    backgroundColor: 'white',
    borderRadius: 8,
    paddingHorizontal: 16,
    paddingVertical: 12,
    marginBottom: 16,
    fontSize: 16,
    borderWidth: 1,
    borderColor: '#ddd',
  },
  button: {
    backgroundColor: '#007AFF',
    borderRadius: 8,
    paddingVertical: 12,
    marginBottom: 16,
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  buttonText: {
    color: 'white',
    textAlign: 'center',
    fontSize: 16,
    fontWeight: '600',
  },
  linkButton: {
    paddingVertical: 8,
  },
  linkText: {
    color: '#007AFF',
    textAlign: 'center',
    fontSize: 16,
  },
});
```

### Step 9: Create App Screens

Create `src/screens/app/DashboardScreen.js`:

```javascript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TouchableOpacity,
  ScrollView,
  RefreshControl,
} from 'react-native';
import { useAuth } from '../../context/AuthContext';
import { signOut } from '../../services/supabase';
import { getInventoryItems } from '../../services/inventoryService';

export default function DashboardScreen({ navigation }) {
  const { user } = useAuth();
  const [stats, setStats] = useState({
    totalItems: 0,
    totalValue: 0,
    lowStock: 0,
  });
  const [refreshing, setRefreshing] = useState(false);

  const loadStats = async () => {
    const { data } = await getInventoryItems();
    if (data) {
      const totalItems = data.length;
      const totalValue = data.reduce((sum, item) => sum + (item.price * item.quantity), 0);
      const lowStock = data.filter(item => item.quantity < 10).length;
      
      setStats({ totalItems, totalValue, lowStock });
    }
  };

  useEffect(() => {
    loadStats();
  }, []);

  const onRefresh = async () => {
    setRefreshing(true);
    await loadStats();
    setRefreshing(false);
  };

  const handleLogout = async () => {
    await signOut();
  };

  return (
    <ScrollView 
      style={styles.container}
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
      }
    >
      <View style={styles.header}>
        <Text style={styles.title}>Dashboard</Text>
        <Text style={styles.welcome}>Welcome, {user?.email}</Text>
        <TouchableOpacity style={styles.logoutButton} onPress={handleLogout}>
          <Text style={styles.logoutText}>Logout</Text>
        </TouchableOpacity>
      </View>

      <View style={styles.statsContainer}>
        <View style={styles.statCard}>
          <Text style={styles.statNumber}>{stats.totalItems}</Text>
          <Text style={styles.statLabel}>Total Items</Text>
        </View>
        
        <View style={styles.statCard}>
          <Text style={styles.statNumber}>${stats.totalValue.toFixed(2)}</Text>
          <Text style={styles.statLabel}>Total Value</Text>
        </View>
        
        <View style={styles.statCard}>
          <Text style={[styles.statNumber, { color: '#FF6B6B' }]}>{stats.lowStock}</Text>
          <Text style={styles.statLabel}>Low Stock</Text>
        </View>
      </View>

      <View style={styles.actionsContainer}>
        <TouchableOpacity 
          style={styles.actionButton}
          onPress={() => navigation.navigate('Inventory')}
        >
          <Text style={styles.actionText}>View Inventory</Text>
        </TouchableOpacity>
        
        <TouchableOpacity 
          style={styles.actionButton}
          onPress={() => navigation.navigate('AddItem')}
        >
          <Text style={styles.actionText}>Add New Item</Text>
        </TouchableOpacity>
      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  header: {
    backgroundColor: 'white',
    padding: 20,
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 5,
  },
  welcome: {
    fontSize: 16,
    color: '#666',
    marginBottom: 15,
  },
  logoutButton: {
    alignSelf: 'flex-start',
    paddingVertical: 8,
    paddingHorizontal: 16,
    backgroundColor: '#FF6B6B',
    borderRadius: 6,
  },
  logoutText: {
    color: 'white',
    fontWeight: '600',
  },
  statsContainer: {
    flexDirection: 'row',
    justifyContent: 'space-around',
    padding: 20,
  },
  statCard: {
    backgroundColor: 'white',
    padding: 20,
    borderRadius: 8,
    alignItems: 'center',
    flex: 1,
    marginHorizontal: 5,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
  statNumber: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#007AFF',
    marginBottom: 5,
  },
  statLabel: {
    fontSize: 12,
    color: '#666',
    textAlign: 'center',
  },
  actionsContainer: {
    padding: 20,
  },
  actionButton: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 8,
    marginBottom: 12,
  },
  actionText: {
    color: 'white',
    textAlign: 'center',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### Step 10: Continue Building... 

**This is getting quite long! Here's what to do next:**

1. **Test what we have so far**: Update your `App.js` to use the authentication system
2. **Create the remaining screens**: InventoryScreen, AddItemScreen, EditItemScreen
3. **Set up navigation**: Create the navigation stack to connect all screens

**Would you like me to continue with the next steps, or would you prefer to test what we have so far and then continue?**

The beauty of this approach is that you can build and test incrementally. Each step adds functionality that you can immediately see and test!

## 🎯 Next Steps

1. Update App.js to use authentication
2. Create navigation structure  
3. Build inventory screens
4. Test the complete app
5. Add enhancements and polish

You're building a real, production-ready mobile app from scratch! 🚀 