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

### Step 10: Complete Navigation Setup and Authentication Flow

**This is getting quite long! But let's complete the full implementation.**

Update your `App.js` to use the authentication system:

```javascript
import React from 'react';
import { AuthProvider } from './src/context/AuthContext';
import AppNavigator from './src/navigation/AppNavigator';

export default function App() {
  return (
    <AuthProvider>
      <AppNavigator />
    </AuthProvider>
  );
}
```

### Step 11: Create Navigation Structure

Create `src/navigation/AppNavigator.js`:

```javascript
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { useAuth } from '../context/AuthContext';
import LoadingScreen from '../screens/LoadingScreen';
import AuthStack from './AuthStack';
import AppStack from './AppStack';

export default function AppNavigator() {
  const { user, loading } = useAuth();

  if (loading) {
    return <LoadingScreen />;
  }

  return (
    <NavigationContainer>
      {user ? <AppStack /> : <AuthStack />}
    </NavigationContainer>
  );
}
```

Create `src/navigation/AuthStack.js`:

```javascript
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import LoginScreen from '../screens/auth/LoginScreen';
import RegisterScreen from '../screens/auth/RegisterScreen';

const Stack = createNativeStackNavigator();

export default function AuthStack() {
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      <Stack.Screen name="Login" component={LoginScreen} />
      <Stack.Screen name="Register" component={RegisterScreen} />
    </Stack.Navigator>
  );
}
```

Create `src/navigation/AppStack.js`:

```javascript
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import DashboardScreen from '../screens/app/DashboardScreen';
import InventoryScreen from '../screens/app/InventoryScreen';
import AddItemScreen from '../screens/app/AddItemScreen';
import EditItemScreen from '../screens/app/EditItemScreen';

const Stack = createNativeStackNavigator();

export default function AppStack() {
  return (
    <Stack.Navigator>
      <Stack.Screen 
        name="Dashboard" 
        component={DashboardScreen}
        options={{ title: 'Inventory Dashboard' }}
      />
      <Stack.Screen 
        name="Inventory" 
        component={InventoryScreen}
        options={{ title: 'All Items' }}
      />
      <Stack.Screen 
        name="AddItem" 
        component={AddItemScreen}
        options={{ title: 'Add New Item' }}
      />
      <Stack.Screen 
        name="EditItem" 
        component={EditItemScreen}
        options={{ title: 'Edit Item' }}
      />
    </Stack.Navigator>
  );
}
```

### Step 12: Create Inventory Screen

Create `src/screens/app/InventoryScreen.js`:

```javascript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  StyleSheet,
  FlatList,
  TouchableOpacity,
  Alert,
  RefreshControl,
  TextInput,
} from 'react-native';
import { getInventoryItems, deleteInventoryItem } from '../../services/inventoryService';

export default function InventoryScreen({ navigation }) {
  const [items, setItems] = useState([]);
  const [filteredItems, setFilteredItems] = useState([]);
  const [loading, setLoading] = useState(true);
  const [refreshing, setRefreshing] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');

  const loadItems = async () => {
    const { data, error } = await getInventoryItems();
    if (error) {
      Alert.alert('Error', error);
    } else {
      setItems(data || []);
      setFilteredItems(data || []);
    }
    setLoading(false);
  };

  useEffect(() => {
    loadItems();
  }, []);

  useEffect(() => {
    // Filter items based on search query
    if (searchQuery.trim() === '') {
      setFilteredItems(items);
    } else {
      const filtered = items.filter(item =>
        item.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
        item.description?.toLowerCase().includes(searchQuery.toLowerCase())
      );
      setFilteredItems(filtered);
    }
  }, [searchQuery, items]);

  const onRefresh = async () => {
    setRefreshing(true);
    await loadItems();
    setRefreshing(false);
  };

  const handleDeleteItem = (item) => {
    Alert.alert(
      'Delete Item',
      `Are you sure you want to delete "${item.name}"?`,
      [
        { text: 'Cancel', style: 'cancel' },
        {
          text: 'Delete',
          style: 'destructive',
          onPress: async () => {
            const { error } = await deleteInventoryItem(item.id);
            if (error) {
              Alert.alert('Error', error);
            } else {
              loadItems();
            }
          },
        },
      ]
    );
  };

  const renderItem = ({ item }) => (
    <View style={styles.itemCard}>
      <View style={styles.itemHeader}>
        <Text style={styles.itemName}>{item.name}</Text>
        <Text style={styles.itemPrice}>${item.price}</Text>
      </View>
      
      {item.description ? (
        <Text style={styles.itemDescription}>{item.description}</Text>
      ) : null}
      
      <View style={styles.itemFooter}>
        <Text style={styles.itemQuantity}>Qty: {item.quantity}</Text>
        <View style={styles.itemActions}>
          <TouchableOpacity
            style={styles.editButton}
            onPress={() => navigation.navigate('EditItem', { item })}
          >
            <Text style={styles.editButtonText}>Edit</Text>
          </TouchableOpacity>
          
          <TouchableOpacity
            style={styles.deleteButton}
            onPress={() => handleDeleteItem(item)}
          >
            <Text style={styles.deleteButtonText}>Delete</Text>
          </TouchableOpacity>
        </View>
      </View>
    </View>
  );

  const renderEmptyState = () => (
    <View style={styles.emptyState}>
      <Text style={styles.emptyStateTitle}>No Items Found</Text>
      <Text style={styles.emptyStateText}>
        {searchQuery ? 'Try adjusting your search' : 'Add your first inventory item to get started!'}
      </Text>
      {!searchQuery && (
        <TouchableOpacity
          style={styles.addButton}
          onPress={() => navigation.navigate('AddItem')}
        >
          <Text style={styles.addButtonText}>Add Item</Text>
        </TouchableOpacity>
      )}
    </View>
  );

  if (loading) {
    return (
      <View style={styles.centered}>
        <Text>Loading...</Text>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <View style={styles.searchContainer}>
        <TextInput
          style={styles.searchInput}
          placeholder="Search items..."
          value={searchQuery}
          onChangeText={setSearchQuery}
        />
      </View>

      <FlatList
        data={filteredItems}
        renderItem={renderItem}
        keyExtractor={(item) => item.id}
        contentContainerStyle={styles.listContainer}
        refreshControl={
          <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
        }
        ListEmptyComponent={renderEmptyState}
      />

      <TouchableOpacity
        style={styles.fab}
        onPress={() => navigation.navigate('AddItem')}
      >
        <Text style={styles.fabText}>+</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  searchContainer: {
    backgroundColor: 'white',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  searchInput: {
    backgroundColor: '#f8f8f8',
    borderRadius: 8,
    paddingHorizontal: 16,
    paddingVertical: 12,
    fontSize: 16,
  },
  listContainer: {
    padding: 16,
    flexGrow: 1,
  },
  itemCard: {
    backgroundColor: 'white',
    borderRadius: 8,
    padding: 16,
    marginBottom: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
  itemHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 8,
  },
  itemName: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#333',
    flex: 1,
  },
  itemPrice: {
    fontSize: 16,
    fontWeight: '600',
    color: '#007AFF',
  },
  itemDescription: {
    fontSize: 14,
    color: '#666',
    marginBottom: 12,
  },
  itemFooter: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  itemQuantity: {
    fontSize: 14,
    color: '#666',
  },
  itemActions: {
    flexDirection: 'row',
  },
  editButton: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 4,
    marginRight: 8,
  },
  editButtonText: {
    color: 'white',
    fontSize: 12,
    fontWeight: '600',
  },
  deleteButton: {
    backgroundColor: '#FF6B6B',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 4,
  },
  deleteButtonText: {
    color: 'white',
    fontSize: 12,
    fontWeight: '600',
  },
  emptyState: {
    alignItems: 'center',
    justifyContent: 'center',
    flex: 1,
    paddingTop: 100,
  },
  emptyStateTitle: {
    fontSize: 20,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  emptyStateText: {
    fontSize: 16,
    color: '#666',
    textAlign: 'center',
    marginBottom: 20,
  },
  addButton: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  addButtonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  fab: {
    position: 'absolute',
    right: 20,
    bottom: 20,
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 4,
    elevation: 5,
  },
  fabText: {
    color: 'white',
    fontSize: 24,
    fontWeight: 'bold',
  },
  centered: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
});
```

### Step 13: Create Add Item Screen

Create `src/screens/app/AddItemScreen.js`:

```javascript
import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TextInput,
  TouchableOpacity,
  ScrollView,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import { addInventoryItem } from '../../services/inventoryService';

export default function AddItemScreen({ navigation }) {
  const [formData, setFormData] = useState({
    name: '',
    description: '',
    quantity: '',
    price: '',
  });
  const [loading, setLoading] = useState(false);

  const handleInputChange = (field, value) => {
    setFormData(prev => ({
      ...prev,
      [field]: value,
    }));
  };

  const validateForm = () => {
    if (!formData.name.trim()) {
      Alert.alert('Error', 'Item name is required');
      return false;
    }

    if (!formData.quantity || isNaN(parseInt(formData.quantity))) {
      Alert.alert('Error', 'Please enter a valid quantity');
      return false;
    }

    if (!formData.price || isNaN(parseFloat(formData.price))) {
      Alert.alert('Error', 'Please enter a valid price');
      return false;
    }

    if (parseInt(formData.quantity) < 0) {
      Alert.alert('Error', 'Quantity cannot be negative');
      return false;
    }

    if (parseFloat(formData.price) < 0) {
      Alert.alert('Error', 'Price cannot be negative');
      return false;
    }

    return true;
  };

  const handleSubmit = async () => {
    if (!validateForm()) return;

    setLoading(true);

    const itemData = {
      name: formData.name.trim(),
      description: formData.description.trim() || null,
      quantity: parseInt(formData.quantity),
      price: parseFloat(formData.price),
    };

    const { data, error } = await addInventoryItem(itemData);
    setLoading(false);

    if (error) {
      Alert.alert('Error', error);
    } else {
      Alert.alert(
        'Success',
        'Item added successfully!',
        [
          {
            text: 'OK',
            onPress: () => navigation.goBack(),
          },
        ]
      );
    }
  };

  return (
    <KeyboardAvoidingView 
      style={styles.container} 
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <ScrollView style={styles.content}>
        <View style={styles.form}>
          <View style={styles.inputGroup}>
            <Text style={styles.label}>Item Name *</Text>
            <TextInput
              style={styles.input}
              placeholder="Enter item name"
              value={formData.name}
              onChangeText={(value) => handleInputChange('name', value)}
            />
          </View>

          <View style={styles.inputGroup}>
            <Text style={styles.label}>Description</Text>
            <TextInput
              style={[styles.input, styles.textArea]}
              placeholder="Enter description (optional)"
              value={formData.description}
              onChangeText={(value) => handleInputChange('description', value)}
              multiline
              numberOfLines={3}
            />
          </View>

          <View style={styles.row}>
            <View style={[styles.inputGroup, styles.halfWidth]}>
              <Text style={styles.label}>Quantity *</Text>
              <TextInput
                style={styles.input}
                placeholder="0"
                value={formData.quantity}
                onChangeText={(value) => handleInputChange('quantity', value)}
                keyboardType="numeric"
              />
            </View>

            <View style={[styles.inputGroup, styles.halfWidth]}>
              <Text style={styles.label}>Price *</Text>
              <TextInput
                style={styles.input}
                placeholder="0.00"
                value={formData.price}
                onChangeText={(value) => handleInputChange('price', value)}
                keyboardType="decimal-pad"
              />
            </View>
          </View>

          <TouchableOpacity
            style={[styles.submitButton, loading && styles.buttonDisabled]}
            onPress={handleSubmit}
            disabled={loading}
          >
            <Text style={styles.submitButtonText}>
              {loading ? 'Adding Item...' : 'Add Item'}
            </Text>
          </TouchableOpacity>
        </View>
      </ScrollView>
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
  },
  form: {
    padding: 20,
  },
  inputGroup: {
    marginBottom: 20,
  },
  label: {
    fontSize: 16,
    fontWeight: '600',
    color: '#333',
    marginBottom: 8,
  },
  input: {
    backgroundColor: 'white',
    borderRadius: 8,
    paddingHorizontal: 16,
    paddingVertical: 12,
    fontSize: 16,
    borderWidth: 1,
    borderColor: '#ddd',
  },
  textArea: {
    height: 80,
    textAlignVertical: 'top',
  },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
  },
  halfWidth: {
    width: '48%',
  },
  submitButton: {
    backgroundColor: '#007AFF',
    borderRadius: 8,
    paddingVertical: 16,
    marginTop: 20,
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  submitButtonText: {
    color: 'white',
    textAlign: 'center',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### Step 14: Create Edit Item Screen

Create `src/screens/app/EditItemScreen.js`:

```javascript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TextInput,
  TouchableOpacity,
  ScrollView,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import { updateInventoryItem } from '../../services/inventoryService';

export default function EditItemScreen({ navigation, route }) {
  const { item } = route.params;
  const [formData, setFormData] = useState({
    name: '',
    description: '',
    quantity: '',
    price: '',
  });
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (item) {
      setFormData({
        name: item.name || '',
        description: item.description || '',
        quantity: item.quantity?.toString() || '',
        price: item.price?.toString() || '',
      });
    }
  }, [item]);

  const handleInputChange = (field, value) => {
    setFormData(prev => ({
      ...prev,
      [field]: value,
    }));
  };

  const validateForm = () => {
    if (!formData.name.trim()) {
      Alert.alert('Error', 'Item name is required');
      return false;
    }

    if (!formData.quantity || isNaN(parseInt(formData.quantity))) {
      Alert.alert('Error', 'Please enter a valid quantity');
      return false;
    }

    if (!formData.price || isNaN(parseFloat(formData.price))) {
      Alert.alert('Error', 'Please enter a valid price');
      return false;
    }

    if (parseInt(formData.quantity) < 0) {
      Alert.alert('Error', 'Quantity cannot be negative');
      return false;
    }

    if (parseFloat(formData.price) < 0) {
      Alert.alert('Error', 'Price cannot be negative');
      return false;
    }

    return true;
  };

  const handleSubmit = async () => {
    if (!validateForm()) return;

    setLoading(true);

    const updates = {
      name: formData.name.trim(),
      description: formData.description.trim() || null,
      quantity: parseInt(formData.quantity),
      price: parseFloat(formData.price),
    };

    const { data, error } = await updateInventoryItem(item.id, updates);
    setLoading(false);

    if (error) {
      Alert.alert('Error', error);
    } else {
      Alert.alert(
        'Success',
        'Item updated successfully!',
        [
          {
            text: 'OK',
            onPress: () => navigation.goBack(),
          },
        ]
      );
    }
  };

  return (
    <KeyboardAvoidingView 
      style={styles.container} 
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <ScrollView style={styles.content}>
        <View style={styles.form}>
          <View style={styles.inputGroup}>
            <Text style={styles.label}>Item Name *</Text>
            <TextInput
              style={styles.input}
              placeholder="Enter item name"
              value={formData.name}
              onChangeText={(value) => handleInputChange('name', value)}
            />
          </View>

          <View style={styles.inputGroup}>
            <Text style={styles.label}>Description</Text>
            <TextInput
              style={[styles.input, styles.textArea]}
              placeholder="Enter description (optional)"
              value={formData.description}
              onChangeText={(value) => handleInputChange('description', value)}
              multiline
              numberOfLines={3}
            />
          </View>

          <View style={styles.row}>
            <View style={[styles.inputGroup, styles.halfWidth]}>
              <Text style={styles.label}>Quantity *</Text>
              <TextInput
                style={styles.input}
                placeholder="0"
                value={formData.quantity}
                onChangeText={(value) => handleInputChange('quantity', value)}
                keyboardType="numeric"
              />
            </View>

            <View style={[styles.inputGroup, styles.halfWidth]}>
              <Text style={styles.label}>Price *</Text>
              <TextInput
                style={styles.input}
                placeholder="0.00"
                value={formData.price}
                onChangeText={(value) => handleInputChange('price', value)}
                keyboardType="decimal-pad"
              />
            </View>
          </View>

          <TouchableOpacity
            style={[styles.submitButton, loading && styles.buttonDisabled]}
            onPress={handleSubmit}
            disabled={loading}
          >
            <Text style={styles.submitButtonText}>
              {loading ? 'Updating Item...' : 'Update Item'}
            </Text>
          </TouchableOpacity>
        </View>
      </ScrollView>
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
  },
  form: {
    padding: 20,
  },
  inputGroup: {
    marginBottom: 20,
  },
  label: {
    fontSize: 16,
    fontWeight: '600',
    color: '#333',
    marginBottom: 8,
  },
  input: {
    backgroundColor: 'white',
    borderRadius: 8,
    paddingHorizontal: 16,
    paddingVertical: 12,
    fontSize: 16,
    borderWidth: 1,
    borderColor: '#ddd',
  },
  textArea: {
    height: 80,
    textAlignVertical: 'top',
  },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
  },
  halfWidth: {
    width: '48%',
  },
  submitButton: {
    backgroundColor: '#007AFF',
    borderRadius: 8,
    paddingVertical: 16,
    marginTop: 20,
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  submitButtonText: {
    color: 'white',
    textAlign: 'center',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### Step 15: Testing Your App

Now that all components are created, let's test the complete application:

1. **Start the app**:
   ```bash
   expo start
   ```

2. **Test Authentication Flow**:
   - Register a new account
   - Check your email for verification (if required)
   - Log in with your credentials
   - Verify the dashboard loads

3. **Test Inventory Features**:
   - Add a new item from the dashboard
   - View all items in the inventory screen
   - Edit an existing item
   - Delete an item
   - Test search functionality

4. **Test Error Handling**:
   - Try submitting forms with invalid data
   - Test without internet connection
   - Try logging in with wrong credentials

### Step 16: Enhanced Features & Improvements

Now let's add some advanced features to make your app even better:

#### A. Add Pull-to-Refresh on Dashboard

Update your `DashboardScreen.js` to include real-time updates:

```javascript
// Add to useEffect in DashboardScreen
useEffect(() => {
  const unsubscribe = navigation.addListener('focus', () => {
    loadStats();
  });
  return unsubscribe;
}, [navigation]);
```

#### B. Add Input Validation Helpers

Create `src/utils/validation.js`:

```javascript
export const validateEmail = (email) => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
};

export const validatePrice = (price) => {
  const numPrice = parseFloat(price);
  return !isNaN(numPrice) && numPrice >= 0;
};

export const validateQuantity = (quantity) => {
  const numQuantity = parseInt(quantity);
  return !isNaN(numQuantity) && numQuantity >= 0;
};

export const formatCurrency = (amount) => {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(amount);
};
```

#### C. Add Loading States and Better Error Handling

Create `src/components/LoadingButton.js`:

```javascript
import React from 'react';
import { TouchableOpacity, Text, ActivityIndicator, StyleSheet } from 'react-native';

export default function LoadingButton({ 
  onPress, 
  title, 
  loading, 
  disabled, 
  style, 
  textStyle 
}) {
  return (
    <TouchableOpacity
      style={[
        styles.button,
        style,
        (loading || disabled) && styles.disabled
      ]}
      onPress={onPress}
      disabled={loading || disabled}
    >
      {loading ? (
        <ActivityIndicator color="white" />
      ) : (
        <Text style={[styles.text, textStyle]}>{title}</Text>
      )}
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  button: {
    backgroundColor: '#007AFF',
    borderRadius: 8,
    paddingVertical: 12,
    paddingHorizontal: 24,
    alignItems: 'center',
    justifyContent: 'center',
  },
  disabled: {
    opacity: 0.6,
  },
  text: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

#### D. Add Image Support (Optional)

To add image support for inventory items:

1. Install image picker:
   ```bash
   expo install expo-image-picker
   ```

2. Update your database schema:
   ```sql
   ALTER TABLE inventory_items ADD COLUMN image_url TEXT;
   ```

3. Add image picker to your Add/Edit screens (this is advanced, implement if time allows)

### Step 17: Production Deployment

When you're ready to deploy your app:

#### A. Build for Production

```bash
# Build for iOS
expo build:ios

# Build for Android
expo build:android

# Or use the new EAS Build (recommended)
npm install -g @expo/eas-cli
eas login
eas build --platform all
```

#### B. Environment Variables

Create `.env` file for production:

```bash
SUPABASE_URL=your_production_supabase_url
SUPABASE_ANON_KEY=your_production_supabase_key
```

Update your `supabase.js` to use environment variables:

```javascript
import { SUPABASE_URL, SUPABASE_ANON_KEY } from '@env';

const supabaseUrl = SUPABASE_URL;
const supabaseAnonKey = SUPABASE_ANON_KEY;
```

### Step 18: Advanced Features (Bonus)

If you want to extend your app further:

#### A. Categories System

Add item categories:

```sql
-- Create categories table
CREATE TABLE categories (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Add category_id to inventory_items
ALTER TABLE inventory_items ADD COLUMN category_id UUID REFERENCES categories(id);
```

#### B. Barcode Scanning

Add barcode scanning with `expo-barcode-scanner`:

```bash
expo install expo-barcode-scanner
```

#### C. Offline Support

Implement offline functionality with local storage:

```bash
expo install @react-native-async-storage/async-storage
```

#### D. Push Notifications

Add low-stock alerts:

```bash
expo install expo-notifications
```

## 🎉 Congratulations!

You've built a complete, production-ready React Native inventory management app! This app includes:

- ✅ User authentication and registration
- ✅ Secure database integration with Supabase
- ✅ CRUD operations for inventory items
- ✅ Real-time data synchronization
- ✅ Professional mobile UI/UX
- ✅ Search and filter functionality
- ✅ Form validation and error handling
- ✅ Navigation between screens
- ✅ Pull-to-refresh and loading states

## 🚀 What You've Learned

- React Native fundamentals
- Navigation with React Navigation
- State management with Context API
- Database integration with Supabase
- Authentication flows
- Form handling and validation
- Mobile UI/UX best practices
- Real-time data synchronization
- Error handling and loading states

## 📚 Next Steps

1. **Deploy your app** to app stores
2. **Add more features** like categories, barcode scanning
3. **Implement offline support** for better user experience
4. **Add analytics** to track app usage
5. **Implement push notifications** for low stock alerts
6. **Add unit tests** to ensure code quality

You now have a solid foundation in React Native development and can build complex mobile applications! 🎯

## 🆘 Troubleshooting

### Common Issues:

1. **Metro bundler errors**: 
   ```bash
   expo start --clear
   ```

2. **Supabase connection issues**:
   - Verify your URL and API key
   - Check your database policies
   - Ensure RLS is properly configured

3. **Navigation errors**:
   - Make sure all screens are properly imported
   - Check that navigation dependencies are installed

4. **Build errors**:
   - Update your dependencies
   - Clear cache and reinstall node_modules

### Getting Help:

- [Expo Documentation](https://docs.expo.dev/)
- [React Navigation Docs](https://reactnavigation.org/)
- [Supabase Documentation](https://supabase.com/docs)
- [React Native Community](https://reactnative.dev/community/overview)

Happy coding! 🚀📱 