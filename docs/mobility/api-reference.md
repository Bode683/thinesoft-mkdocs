# Mobility API Reference

Technical reference for the Digital Mobility Platform API and integration documentation.

## Overview

The Mobility Platform uses **Supabase** as its backend-as-a-service, providing:

- RESTful API endpoints
- Real-time subscriptions
- PostgreSQL database
- Row-level security
- Authentication and authorization

## Base URL

```
https://your-project.supabase.co
```

## Authentication

All API requests require authentication using Supabase JWT tokens.

### Authentication Flow

```typescript
import { supabase } from '@/lib/supabase'

// Sign up
const { data, error } = await supabase.auth.signUp({
  email: 'user@example.com',
  password: 'secure-password'
})

// Sign in
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password'
})

// Get session
const { data: { session } } = await supabase.auth.getSession()

// Sign out
await supabase.auth.signOut()
```

### Headers

Include the auth token in requests:

```
Authorization: Bearer <your-jwt-token>
apikey: <your-supabase-anon-key>
```

## API Endpoints

### Authentication

#### Sign Up

```http
POST /auth/v1/signup
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "created_at": "2024-01-01T00:00:00Z"
  },
  "session": {
    "access_token": "jwt-token",
    "refresh_token": "refresh-token"
  }
}
```

#### Sign In

```http
POST /auth/v1/token?grant_type=password
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}
```

### Profiles

#### Get User Profile

```typescript
const { data, error } = await supabase
  .from('profiles')
  .select('*')
  .eq('id', userId)
  .single()
```

**SQL Schema:**
```sql
CREATE TABLE profiles (
  id UUID REFERENCES auth.users PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  phone_number TEXT,
  avatar_url TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### Update Profile

```typescript
const { data, error } = await supabase
  .from('profiles')
  .update({
    full_name: 'John Doe',
    phone_number: '+1234567890'
  })
  .eq('id', userId)
```

### Saved Addresses

#### List Saved Addresses

```typescript
const { data, error } = await supabase
  .from('saved_addresses')
  .select('*')
  .eq('user_id', userId)
  .order('created_at', { ascending: false })
```

**Response:**
```json
[
  {
    "id": "uuid",
    "user_id": "uuid",
    "label": "home",
    "address": "123 Main St, City, State",
    "latitude": 40.7128,
    "longitude": -74.0060,
    "created_at": "2024-01-01T00:00:00Z"
  }
]
```

#### Add Saved Address

```typescript
const { data, error } = await supabase
  .from('saved_addresses')
  .insert({
    user_id: userId,
    label: 'home',
    address: '123 Main St',
    latitude: 40.7128,
    longitude: -74.0060
  })
```

#### Update Saved Address

```typescript
const { data, error } = await supabase
  .from('saved_addresses')
  .update({ address: 'New Address' })
  .eq('id', addressId)
```

#### Delete Saved Address

```typescript
const { data, error } = await supabase
  .from('saved_addresses')
  .delete()
  .eq('id', addressId)
```

### Trips

#### Get Trip History

```typescript
const { data, error } = await supabase
  .from('trips')
  .select('*')
  .eq('user_id', userId)
  .order('created_at', { ascending: false })
  .limit(50)
```

**Response:**
```json
[
  {
    "id": "uuid",
    "user_id": "uuid",
    "driver_id": "uuid",
    "pickup_address": "123 Start St",
    "pickup_lat": 40.7128,
    "pickup_lng": -74.0060,
    "dropoff_address": "456 End Ave",
    "dropoff_lat": 40.7580,
    "dropoff_lng": -73.9855,
    "distance": 5.2,
    "duration": 15,
    "fare": 12.50,
    "status": "completed",
    "vehicle_type": "economy",
    "created_at": "2024-01-01T10:00:00Z",
    "completed_at": "2024-01-01T10:15:00Z"
  }
]
```

#### Create Trip

```typescript
const { data, error } = await supabase
  .from('trips')
  .insert({
    user_id: userId,
    pickup_address: '123 Start St',
    pickup_lat: 40.7128,
    pickup_lng: -74.0060,
    dropoff_address: '456 End Ave',
    dropoff_lat: 40.7580,
    dropoff_lng: -73.9855,
    vehicle_type: 'economy',
    status: 'pending'
  })
```

#### Update Trip Status

```typescript
const { data, error } = await supabase
  .from('trips')
  .update({
    status: 'completed',
    completed_at: new Date().toISOString()
  })
  .eq('id', tripId)
```

### Payment Methods

#### List Payment Methods

```typescript
const { data, error } = await supabase
  .from('payment_methods')
  .select('*')
  .eq('user_id', userId)
```

#### Add Payment Method

```typescript
const { data, error } = await supabase
  .from('payment_methods')
  .insert({
    user_id: userId,
    type: 'card',
    last_four: '4242',
    card_brand: 'visa',
    is_default: true
  })
```

#### Set Default Payment Method

```typescript
// First, unset all as default
await supabase
  .from('payment_methods')
  .update({ is_default: false })
  .eq('user_id', userId)

// Then set the chosen one as default
await supabase
  .from('payment_methods')
  .update({ is_default: true })
  .eq('id', paymentMethodId)
```

## Real-Time Subscriptions

### Subscribe to Trip Updates

```typescript
const subscription = supabase
  .channel('trip-updates')
  .on(
    'postgres_changes',
    {
      event: 'UPDATE',
      schema: 'public',
      table: 'trips',
      filter: `id=eq.${tripId}`
    },
    (payload) => {
      console.log('Trip updated:', payload.new)
      // Update UI with new trip status
    }
  )
  .subscribe()

// Unsubscribe when done
subscription.unsubscribe()
```

### Subscribe to Driver Location

```typescript
const driverChannel = supabase
  .channel(`driver:${driverId}`)
  .on('broadcast', { event: 'location' }, (payload) => {
    const { latitude, longitude } = payload
    // Update driver marker on map
  })
  .subscribe()
```

## Mapbox API Integration

### Geocoding API

#### Forward Geocoding (Address to Coordinates)

```typescript
const forwardGeocode = async (address: string) => {
  const url = `https://api.mapbox.com/geocoding/v5/mapbox.places/${encodeURIComponent(address)}.json`
  const params = new URLSearchParams({
    access_token: MAPBOX_TOKEN,
    limit: '5',
    types: 'address,poi'
  })

  const response = await fetch(`${url}?${params}`)
  const data = await response.json()

  return data.features.map(feature => ({
    address: feature.place_name,
    coordinates: feature.geometry.coordinates
  }))
}
```

#### Reverse Geocoding (Coordinates to Address)

```typescript
const reverseGeocode = async (latitude: number, longitude: number) => {
  const url = `https://api.mapbox.com/geocoding/v5/mapbox.places/${longitude},${latitude}.json`
  const params = new URLSearchParams({
    access_token: MAPBOX_TOKEN,
    types: 'address'
  })

  const response = await fetch(`${url}?${params}`)
  const data = await response.json()

  return data.features[0]?.place_name || 'Unknown location'
}
```

### Directions API

#### Get Route Between Points

```typescript
const getRoute = async (
  startCoords: [number, number],
  endCoords: [number, number]
) => {
  const coordinates = `${startCoords[0]},${startCoords[1]};${endCoords[0]},${endCoords[1]}`
  const url = `https://api.mapbox.com/directions/v5/mapbox/driving/${coordinates}`

  const params = new URLSearchParams({
    access_token: MAPBOX_TOKEN,
    geometries: 'geojson',
    overview: 'full',
    steps: 'true'
  })

  const response = await fetch(`${url}?${params}`)
  const data = await response.json()

  return {
    route: data.routes[0].geometry,
    distance: data.routes[0].distance,  // meters
    duration: data.routes[0].duration,  // seconds
    steps: data.routes[0].legs[0].steps
  }
}
```

## React Query Hooks

### useProfile

```typescript
import { useQuery } from '@tanstack/react-query'

export const useProfile = (userId: string) => {
  return useQuery({
    queryKey: ['profile', userId],
    queryFn: async () => {
      const { data, error } = await supabase
        .from('profiles')
        .select('*')
        .eq('id', userId)
        .single()

      if (error) throw error
      return data
    }
  })
}

// Usage
const { data: profile, isLoading, error } = useProfile(userId)
```

### useSavedAddresses

```typescript
export const useSavedAddresses = (userId: string) => {
  return useQuery({
    queryKey: ['addresses', userId],
    queryFn: async () => {
      const { data, error } = await supabase
        .from('saved_addresses')
        .select('*')
        .eq('user_id', userId)

      if (error) throw error
      return data
    }
  })
}
```

### useTrips

```typescript
export const useTrips = (userId: string) => {
  return useQuery({
    queryKey: ['trips', userId],
    queryFn: async () => {
      const { data, error } = await supabase
        .from('trips')
        .select('*')
        .eq('user_id', userId)
        .order('created_at', { ascending: false })

      if (error) throw error
      return data
    }
  })
}
```

### useCreateTrip (Mutation)

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'

export const useCreateTrip = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (tripData: TripCreateInput) => {
      const { data, error } = await supabase
        .from('trips')
        .insert(tripData)
        .select()
        .single()

      if (error) throw error
      return data
    },
    onSuccess: () => {
      // Invalidate and refetch trips
      queryClient.invalidateQueries({ queryKey: ['trips'] })
    }
  })
}

// Usage
const createTrip = useCreateTrip()

const handleBookRide = async () => {
  try {
    const trip = await createTrip.mutateAsync({
      user_id: userId,
      pickup_address: '...',
      // ... other trip data
    })
    console.log('Trip created:', trip)
  } catch (error) {
    console.error('Failed to create trip:', error)
  }
}
```

## Type Definitions

### TypeScript Interfaces

```typescript
// User Profile
interface Profile {
  id: string
  email: string
  full_name: string | null
  phone_number: string | null
  avatar_url: string | null
  created_at: string
  updated_at: string
}

// Saved Address
interface SavedAddress {
  id: string
  user_id: string
  label: string  // 'home' | 'work' | custom
  address: string
  latitude: number
  longitude: number
  created_at: string
}

// Trip
interface Trip {
  id: string
  user_id: string
  driver_id: string | null
  pickup_address: string
  pickup_lat: number
  pickup_lng: number
  dropoff_address: string
  dropoff_lat: number
  dropoff_lng: number
  distance: number  // kilometers
  duration: number  // minutes
  fare: number
  status: TripStatus
  vehicle_type: VehicleType
  created_at: string
  completed_at: string | null
}

// Trip Status
type TripStatus =
  | 'pending'
  | 'driver_assigned'
  | 'driver_arriving'
  | 'driver_arrived'
  | 'in_progress'
  | 'completed'
  | 'cancelled'

// Vehicle Type
type VehicleType =
  | 'economy'
  | 'comfort'
  | 'premium'
  | 'xl'

// Payment Method
interface PaymentMethod {
  id: string
  user_id: string
  type: 'card' | 'wallet' | 'cash'
  last_four: string | null
  card_brand: string | null
  is_default: boolean
  created_at: string
}
```

## Error Handling

### API Errors

```typescript
interface APIError {
  message: string
  code?: string
  details?: any
}

// Example error handling
try {
  const { data, error } = await supabase
    .from('trips')
    .select('*')

  if (error) throw error

  return data
} catch (error: any) {
  if (error.code === 'PGRST116') {
    // No rows returned
    console.log('No trips found')
  } else if (error.message) {
    console.error('Error:', error.message)
  } else {
    console.error('Unknown error occurred')
  }
}
```

### Network Error Handling

```typescript
import axios from 'axios'
import axiosRetry from 'axios-retry'

// Configure automatic retries
axiosRetry(axios, {
  retries: 3,
  retryDelay: axiosRetry.exponentialDelay,
  retryCondition: (error) => {
    return axiosRetry.isNetworkOrIdempotentRequestError(error) ||
           error.response?.status === 429  // Rate limit
  }
})
```

## Rate Limiting

Supabase has default rate limits:

- **Anonymous requests**: 30 requests per minute
- **Authenticated requests**: 60 requests per minute
- **Realtime connections**: 100 concurrent connections

Handle rate limits:

```typescript
const handleRateLimit = async (fn: () => Promise<any>) => {
  try {
    return await fn()
  } catch (error: any) {
    if (error.status === 429) {
      // Wait and retry
      await new Promise(resolve => setTimeout(resolve, 1000))
      return await fn()
    }
    throw error
  }
}
```

## Best Practices

### 1. Use React Query for Caching

```typescript
// ✅ Good - Cached and automatically refetched
const { data } = useQuery({
  queryKey: ['profile'],
  queryFn: fetchProfile,
  staleTime: 5 * 60 * 1000  // 5 minutes
})

// ❌ Bad - Fetches on every render
const [profile, setProfile] = useState(null)
useEffect(() => {
  fetchProfile().then(setProfile)
}, [])
```

### 2. Implement Optimistic Updates

```typescript
const updateProfile = useMutation({
  mutationFn: (updates) => supabase.from('profiles').update(updates),
  onMutate: async (updates) => {
    // Cancel outgoing queries
    await queryClient.cancelQueries({ queryKey: ['profile'] })

    // Snapshot previous value
    const previous = queryClient.getQueryData(['profile'])

    // Optimistically update
    queryClient.setQueryData(['profile'], (old) => ({
      ...old,
      ...updates
    }))

    return { previous }
  },
  onError: (err, variables, context) => {
    // Rollback on error
    queryClient.setQueryData(['profile'], context.previous)
  }
})
```

### 3. Handle Offline State

```typescript
import NetInfo from '@react-native-community/netinfo'

const { data, isLoading } = useQuery({
  queryKey: ['trips'],
  queryFn: fetchTrips,
  enabled: isOnline,  // Only fetch when online
  networkMode: 'offlineFirst'  // Serve cached data when offline
})
```

### 4. Secure API Keys

```typescript
// ✅ Good - Use environment variables
const SUPABASE_URL = process.env.EXPO_PUBLIC_SUPABASE_URL

// ❌ Bad - Hardcoded keys
const SUPABASE_URL = 'https://abc123.supabase.co'
```

## Testing

### Mock Supabase Client

```typescript
import { createClient } from '@supabase/supabase-js'

// Mock for testing
jest.mock('@/lib/supabase', () => ({
  supabase: {
    from: jest.fn(() => ({
      select: jest.fn(() => ({
        eq: jest.fn(() => Promise.resolve({ data: [], error: null }))
      }))
    }))
  }
}))
```

## Next Steps

- [Installation](installation.md) - Set up the platform
- [Configuration](configuration.md) - Configure backend services
- [Features](features.md) - Explore platform capabilities
- [User Guide](user-guide.md) - Learn how to use the app
