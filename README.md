import React, { createContext, useContext, useState, useEffect } from 'react';
import { AlertCircle, CheckCircle, X, Loader2, Calendar, Ticket, Users, TrendingUp, Plus, Edit2, Trash2, LogOut, User } from 'lucide-react';

// Types
interface User {
  id: string;
  email: string;
  name: string;
  role: 'user' | 'admin';
}

interface Event {
  id: string;
  name: string;
  description: string;
  date: string;
  location: string;
  totalCapacity: number;
  availableTickets: number;
  price: number;
}

interface Booking {
  id: string;
  eventId: string;
  eventName: string;
  userId: string;
  userName: string;
  ticketCount: number;
  bookingDate: string;
  totalPrice: number;
}

interface AuthContextType {
  user: User | null;
  token: string | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  isAuthenticated: boolean;
}

interface ToastContextType {
  showToast: (message: string, type: 'success' | 'error') => void;
}

interface LoaderContextType {
  isLoading: boolean;
  setIsLoading: (loading: boolean) => void;
}

// Contexts
const AuthContext = createContext<AuthContextType | null>(null);
const ToastContext = createContext<ToastContextType | null>(null);
const LoaderContext = createContext<LoaderContextType | null>(null);

// Custom Hooks
const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
};

const useToast = () => {
  const context = useContext(ToastContext);
  if (!context) throw new Error('useToast must be used within ToastProvider');
  return context;
};

const useLoader = () => {
  const context = useContext(LoaderContext);
  if (!context) throw new Error('useLoader must be used within LoaderProvider');
  return context;
};

// API Handler with retry logic
const apiCall = async (endpoint: string, options: RequestInit = {}, retries = 3): Promise<any> => {
  const baseURL = 'https://api.example.com'; // Replace with actual backend URL
  
  for (let i = 0; i < retries; i++) {
    try {
      const response = await fetch(`${baseURL}${endpoint}`, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          ...options.headers,
        },
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message || 'API Error');
      }

      return await response.json();
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
};

// Providers
const ToastProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [toast, setToast] = useState<{ message: string; type: 'success' | 'error' } | null>(null);

  const showToast = (message: string, type: 'success' | 'error') => {
    setToast({ message, type });
    setTimeout(() => setToast(null), 4000);
  };

  return (
    <ToastContext.Provider value={{ showToast }}>
      {children}
      {toast && (
        <div className="fixed top-4 right-4 z-50 animate-slide-in">
          <div className={`flex items-center gap-3 px-4 py-3 rounded-lg shadow-lg ${
            toast.type === 'success' ? 'bg-green-500' : 'bg-red-500'
          } text-white min-w-[300px]`}>
            {toast.type === 'success' ? <CheckCircle size={20} /> : <AlertCircle size={20} />}
            <span className="flex-1">{toast.message}</span>
            <button onClick={() => setToast(null)} className="hover:opacity-80">
              <X size={18} />
            </button>
          </div>
        </div>
      )}
    </ToastContext.Provider>
  );
};

const LoaderProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [isLoading, setIsLoading] = useState(false);

  return (
    <LoaderContext.Provider value={{ isLoading, setIsLoading }}>
      {children}
      {isLoading && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
          <div className="bg-white rounded-lg p-6 flex items-center gap-3">
            <Loader2 className="animate-spin text-blue-600" size={24} />
            <span className="text-gray-700 font-medium">Loading...</span>
          </div>
        </div>
      )}
    </LoaderContext.Provider>
  );
};

const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [token, setToken] = useState<string | null>(localStorage.getItem('token'));
  const { showToast } = useToast();
  const { setIsLoading } = useLoader();

  useEffect(() => {
    if (token) {
      validateToken();
    }
  }, []);

  const validateToken = async () => {
    try {
      const data = await apiCall('/auth/validate', {
        headers: { Authorization: `Bearer ${token}` }
      });
      setUser(data.user);
    } catch (error) {
      logout();
    }
  };

  const login = async (email: string, password: string) => {
    setIsLoading(true);
    try {
      const data = await apiCall('/auth/login', {
        method: 'POST',
        body: JSON.stringify({ email, password })
      });
      
      setToken(data.token);
      setUser(data.user);
      localStorage.setItem('token', data.token);
      showToast('Login successful!', 'success');
    } catch (error: any) {
      showToast(error.message || 'Login failed', 'error');
      throw error;
    } finally {
      setIsLoading(false);
    }
  };

  const logout = () => {
    setUser(null);
    setToken(null);
    localStorage.removeItem('token');
    showToast('Logged out successfully', 'success');
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, isAuthenticated: !!token }}>
      {children}
    </AuthContext.Provider>
  );
};

// Components
const LoginPage: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<{ email?: string; password?: string }>({});
  const { login } = useAuth();

  const validate = () => {
    const newErrors: { email?: string; password?: string } = {};
    if (!email) newErrors.email = 'Email is required';
    else if (!/\S+@\S+\.\S+/.test(email)) newErrors.email = 'Email is invalid';
    if (!password) newErrors.password = 'Password is required';
    else if (password.length < 6) newErrors.password = 'Password must be at least 6 characters';
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!validate()) return;
    
    try {
      await login(email, password);
    } catch (error) {
      // Error handled in login function
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 flex items-center justify-center p-4">
      <div className="bg-white rounded-2xl shadow-xl p-8 w-full max-w-md">
        <div className="text-center mb-8">
          <Ticket className="mx-auto text-blue-600 mb-4" size={48} />
          <h1 className="text-3xl font-bold text-gray-800">Welcome Back</h1>
          <p className="text-gray-600 mt-2">Sign in to book your tickets</p>
        </div>

        <form onSubmit={handleSubmit} className="space-y-5">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">Email</label>
            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none"
              placeholder="your@email.com"
            />
            {errors.email && <p className="text-red-500 text-sm mt-1">{errors.email}</p>}
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">Password</label>
            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none"
              placeholder="••••••••"
            />
            {errors.password && <p className="text-red-500 text-sm mt-1">{errors.password}</p>}
          </div>

          <button
            type="submit"
            className="w-full bg-blue-600 text-white py-3 rounded-lg font-semibold hover:bg-blue-700 transition-colors"
          >
            Sign In
          </button>
        </form>

        <div className="mt-6 p-4 bg-gray-50 rounded-lg">
          <p className="text-sm text-gray-600 mb-2">Demo Credentials:</p>
          <p className="text-xs text-gray-500">User: user@demo.com / pass123</p>
          <p className="text-xs text-gray-500">Admin: admin@demo.com / admin123</p>
        </div>
      </div>
    </div>
  );
};

const UserHome: React.FC = () => {
  const [events, setEvents] = useState<Event[]>([]);
  const [selectedEvent, setSelectedEvent] = useState<Event | null>(null);
  const [ticketCount, setTicketCount] = useState(1);
  const { token } = useAuth();
  const { showToast } = useToast();
  const { setIsLoading } = useLoader();

  useEffect(() => {
    fetchEvents();
  }, []);

  const fetchEvents = async () => {
    setIsLoading(true);
    try {
      const data = await apiCall('/events', {
        headers: { Authorization: `Bearer ${token}` }
      });
      setEvents(data.events || []);
    } catch (error: any) {
      showToast(error.message || 'Failed to load events', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  const handleBooking = async () => {
    if (!selectedEvent) return;
    
    if (ticketCount > selectedEvent.availableTickets) {
      showToast('Not enough tickets available', 'error');
      return;
    }

    setIsLoading(true);
    try {
      await apiCall('/bookings', {
        method: 'POST',
        headers: { Authorization: `Bearer ${token}` },
        body: JSON.stringify({ eventId: selectedEvent.id, ticketCount })
      });
      showToast('Booking successful!', 'success');
      setSelectedEvent(null);
      fetchEvents();
    } catch (error: any) {
      showToast(error.message || 'Booking failed', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold text-gray-800">Available Events</h1>
      
      {events.length === 0 ? (
        <div className="text-center py-12 bg-white rounded-lg">
          <Calendar size={48} className="mx-auto text-gray-400 mb-4" />
          <p className="text-gray-600">No events available at the moment</p>
        </div>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          {events.map((event) => (
            <div key={event.id} className="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow">
              <h3 className="text-xl font-bold text-gray-800 mb-2">{event.name}</h3>
              <p className="text-gray-600 text-sm mb-4">{event.description}</p>
              
              <div className="space-y-2 mb-4">
                <div className="flex items-center text-sm text-gray-700">
                  <Calendar size={16} className="mr-2" />
                  {new Date(event.date).toLocaleDateString()}
                </div>
                <div className="flex items-center text-sm text-gray-700">
                  <Ticket size={16} className="mr-2" />
                  {event.availableTickets} / {event.totalCapacity} available
                </div>
                <div className="text-lg font-bold text-blue-600">
                  ${event.price} per ticket
                </div>
              </div>

              <button
                onClick={() => setSelectedEvent(event)}
                disabled={event.availableTickets === 0}
                className={`w-full py-2 rounded-lg font-semibold transition-colors ${
                  event.availableTickets === 0
                    ? 'bg-gray-300 text-gray-500 cursor-not-allowed'
                    : 'bg-blue-600 text-white hover:bg-blue-700'
                }`}
              >
                {event.availableTickets === 0 ? 'Sold Out' : 'Book Now'}
              </button>
            </div>
          ))}
        </div>
      )}

      {selectedEvent && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-40 p-4">
          <div className="bg-white rounded-lg p-6 max-w-md w-full">
            <h2 className="text-2xl font-bold mb-4">{selectedEvent.name}</h2>
            
            <div className="mb-4">
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Number of Tickets
              </label>
              <input
                type="number"
                min="1"
                max={selectedEvent.availableTickets}
                value={ticketCount}
                onChange={(e) => setTicketCount(Math.max(1, parseInt(e.target.value) || 1))}
                className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
              />
            </div>

            <div className="mb-6 p-4 bg-gray-50 rounded-lg">
              <div className="flex justify-between mb-2">
                <span className="text-gray-600">Price per ticket:</span>
                <span className="font-semibold">${selectedEvent.price}</span>
              </div>
              <div className="flex justify-between text-lg font-bold">
                <span>Total:</span>
                <span className="text-blue-600">${selectedEvent.price * ticketCount}</span>
              </div>
            </div>

            <div className="flex gap-3">
              <button
                onClick={() => setSelectedEvent(null)}
                className="flex-1 px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors"
              >
                Cancel
              </button>
              <button
                onClick={handleBooking}
                className="flex-1 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors"
              >
                Confirm Booking
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

const UserBookings: React.FC = () => {
  const [bookings, setBookings] = useState<Booking[]>([]);
  const { token } = useAuth();
  const { showToast } = useToast();
  const { setIsLoading } = useLoader();

  useEffect(() => {
    fetchBookings();
  }, []);

  const fetchBookings = async () => {
    setIsLoading(true);
    try {
      const data = await apiCall('/bookings/my-bookings', {
        headers: { Authorization: `Bearer ${token}` }
      });
      setBookings(data.bookings || []);
    } catch (error: any) {
      showToast(error.message || 'Failed to load bookings', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold text-gray-800">My Bookings</h1>

      {bookings.length === 0 ? (
        <div className="text-center py-12 bg-white rounded-lg">
          <Ticket size={48} className="mx-auto text-gray-400 mb-4" />
          <p className="text-gray-600">You haven't made any bookings yet</p>
        </div>
      ) : (
        <div className="space-y-4">
          {bookings.map((booking) => (
            <div key={booking.id} className="bg-white rounded-lg shadow-md p-6">
              <div className="flex justify-between items-start mb-4">
                <div>
                  <h3 className="text-xl font-bold text-gray-800">{booking.eventName}</h3>
                  <p className="text-sm text-gray-600">
                    Booked on {new Date(booking.bookingDate).toLocaleDateString()}
                  </p>
                </div>
                <div className="text-right">
                  <div className="text-2xl font-bold text-blue-600">${booking.totalPrice}</div>
                  <div className="text-sm text-gray-600">{booking.ticketCount} tickets</div>
                </div>
              </div>
              <div className="flex items-center text-sm text-gray-700">
                <Ticket size={16} className="mr-2" />
                Booking ID: {booking.id}
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

const AdminDashboard: React.FC = () => {
  const [stats, setStats] = useState({
    totalEvents: 0,
    totalBookings: 0,
    totalRevenue: 0,
    totalUsers: 0
  });
  const { token } = useAuth();
  const { showToast } = useToast();
  const { setIsLoading } = useLoader();

  useEffect(() => {
    fetchStats();
  }, []);

  const fetchStats = async () => {
    setIsLoading(true);
    try {
      const data = await apiCall('/admin/stats', {
        headers: { Authorization: `Bearer ${token}` }
      });
      setStats(data.stats || stats);
    } catch (error: any) {
      showToast(error.message || 'Failed to load statistics', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  const statCards = [
    { label: 'Total Events', value: stats.totalEvents, icon: Calendar, color: 'bg-blue-500' },
    { label: 'Total Bookings', value: stats.totalBookings, icon: Ticket, color: 'bg-green-500' },
    { label: 'Total Revenue', value: `$${stats.totalRevenue}`, icon: TrendingUp, color: 'bg-purple-500' },
    { label: 'Total Users', value: stats.totalUsers, icon: Users, color: 'bg-orange-500' }
  ];

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold text-gray-800">Admin Dashboard</h1>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        {statCards.map((stat, index) => (
          <div key={index} className="bg-white rounded-lg shadow-md p-6">
            <div className="flex items-center justify-between mb-4">
              <div className={`${stat.color} p-3 rounded-lg`}>
                <stat.icon size={24} className="text-white" />
              </div>
            </div>
            <div className="text-3xl font-bold text-gray-800 mb-1">{stat.value}</div>
            <div className="text-sm text-gray-600">{stat.label}</div>
          </div>
        ))}
      </div>
    </div>
  );
};

const AdminEvents: React.FC = () => {
  const [events, setEvents] = useState<Event[]>([]);
  const [showModal, setShowModal] = useState(false);
  const [editingEvent, setEditingEvent] = useState<Event | null>(null);
  const [formData, setFormData] = useState({
    name: '',
    description: '',
    date: '',
    location: '',
    totalCapacity: 0,
    price: 0
  });
  const { token } = useAuth();
  const { showToast } = useToast();
  const { setIsLoading } = useLoader();

  useEffect(() => {
    fetchEvents();
  }, []);

  const fetchEvents = async () => {
    setIsLoading(true);
    try {
      const data = await apiCall('/admin/events', {
        headers: { Authorization: `Bearer ${token}` }
      });
      setEvents(data.events || []);
    } catch (error: any) {
      showToast(error.message || 'Failed to load events', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsLoading(true);

    try {
      const url = editingEvent ? `/admin/events/${editingEvent.id}` : '/admin/events';
      const method = editingEvent ? 'PUT' : 'POST';

      await apiCall(url, {
        method,
        headers: { Authorization: `Bearer ${token}` },
        body: JSON.stringify(formData)
      });

      showToast(`Event ${editingEvent ? 'updated' : 'created'} successfully!`, 'success');
      setShowModal(false);
      resetForm();
      fetchEvents();
    } catch (error: any) {
      showToast(error.message || 'Operation failed', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  const handleDelete = async (id: string) => {
    if (!confirm('Are you sure you want to delete this event?')) return;

    setIsLoading(true);
    try {
      await apiCall(`/admin/events/${id}`, {
        method: 'DELETE',
        headers: { Authorization: `Bearer ${token}` }
      });
      showToast('Event deleted successfully!', 'success');
      fetchEvents();
    } catch (error: any) {
      showToast(error.message || 'Delete failed', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  const resetForm = () => {
    setFormData({ name: '', description: '', date: '', location: '', totalCapacity: 0, price: 0 });
    setEditingEvent(null);
  };

  const openEditModal = (event: Event) => {
    setEditingEvent(event);
    setFormData({
      name: event.name,
      description: event.description,
      date: event.date,
      location: event.location,
      totalCapacity: event.totalCapacity,
      price: event.price
    });
    setShowModal(true);
  };

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-3xl font-bold text-gray-800">Manage Events</h1>
        <button
          onClick={() => { resetForm(); setShowModal(true); }}
          className="flex items-center gap-2 bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700 transition-colors"
        >
          <Plus size={20} />
          Add Event
        </button>
      </div>

      {events.length === 0 ? (
        <div className="text-center py-12 bg-white rounded-lg">
          <Calendar size={48} className="mx-auto text-gray-400 mb-4" />
          <p className="text-gray-600">No events created yet</p>
        </div>
      ) : (
        <div className="space-y-4">
          {events.map((event) => (
            <div key={event.id} className="bg-white rounded-lg shadow-md p-6">
              <div className="flex justify-between items-start">
                <div className="flex-1">
                  <h3 className="text-xl font-bold text-gray-800 mb-2">{event.name}</h3>
                  <p className="text-gray-600 text-sm mb-3">{event.description}</p>
                  <div className="grid grid-cols-2 gap-4">
                    <div className="text-sm text-gray-700">
                      <span className="font-medium">Date:</span> {new Date(event.date).toLocaleDateString()}
                    </div>
                    <div className="text-sm text-gray-700">
                      <span className="font-medium">Location:</span> {event.location}
                    </div>
                    <div className="text-sm text-gray-700">
                      <span className="font-medium">Capacity:</span> {event.availableTickets} / {event.totalCapacity}
                    </div>
                    <div className="text-sm text-gray-700">
                      <span className="font-medium">Price:</span> ${event.price}
                    </div>
                  </div>
                </div>
                <div className="flex gap-2 ml-4">
                  <button
                    onClick={() => openEditModal(event)}
                    className="p-2 text-blue-600 hover:bg-blue-50 rounded-lg transition-colors"
                  >
                    <Edit2 size={20} />
                  </button>
                  <button
                    onClick={() => handleDelete(event.id)}
                    className="p-2 text-red-600 hover:bg-red-50 rounded-lg transition-colors"
                  >
                    <Trash2 size={20} />
                  </button>
                </div>
              </div>
            </div>
          ))}
        </div>
      )}

      {showModal && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-40 p-4">
          <div className="bg-white rounded-lg p-6 max-w-2xl w-full max-h-[90vh] overflow-y-auto">
            <h2 className="text-2xl font-bold mb-6">
              {editingEvent ? 'Edit Event' : 'Create New Event'}
            </h2>

            <form onSubmit={handleSubmit} className="space-y-4">
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-2">Event Name</label>
                <input
                  type="text"
                  required
                  value={formData.name}
                  onChange={(e) => setFormData({ ...formData, name: e.target.value })}
                  className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
                />
              </div>

              <div>
                <label className="block text-sm font-medium text-gray-700 mb-2">Description</label>
                <textarea
                  required
                  value={formData.description}
                  onChange={(e) => setFormData({ ...formData, description: e.target.value })}
                  className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
                  rows={3}
                />
              </div>

              <div className="grid grid-cols-2 gap-4">
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">Date</label>
                  <input
                    type="date"
                    required
                    value={formData.date}
                    onChange={(e) => setFormData({ ...formData, date: e.target.value })}
                    className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
                  />
                </div>

                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">Location</label>
                  <input
                    type="text"
                    required
                    value={formData.location}
                    onChange={(e) => setFormData({ ...formData, location: e.target.value })}
                    className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
                  />
                </div>

                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">Total Capacity</label>
                  <input
                    type="number"
                    required
                    min="1"
                    value={formData.totalCapacity}
                    onChange={(e) => setFormData({ ...formData, totalCapacity: parseInt(e.target.value) })}
                    className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
                  />
                </div>

                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">Price ($)</label>
                  <input
                    type="number"
                    required
                    min="0"
                    step="0.01"
                    value={formData.price}
                    onChange={(e) => setFormData({ ...formData, price: parseFloat(e.target.value) })}
                    className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
                  />
                </div>
              </div>

              <div className="flex gap-3 pt-4">
                <button
                  type="button"
                  onClick={() => { setShowModal(false); resetForm(); }}
                  className="flex-1 px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors"
                >
                  Cancel
                </button>
                <button
                  type="submit"
                  className="flex-1 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors"
                >
                  {editingEvent ? 'Update Event' : 'Create Event'}
                </button>
              </div>
            </form>
          </div>
        </div>
      )}
    </div>
  );
};

const AdminBookings: React.FC = () => {
  const [bookings, setBookings] = useState<Booking[]>([]);
  const { token } = useAuth();
  const { showToast } = useToast();
  const { setIsLoading } = useLoader();

  useEffect(() => {
    fetchBookings();
  }, []);

  const fetchBookings = async () => {
    setIsLoading(true);
    try {
      const data = await apiCall('/admin/bookings', {
        headers: { Authorization: `Bearer ${token}` }
      });
      setBookings(data.bookings || []);
    } catch (error: any) {
      showToast(error.message || 'Failed to load bookings', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold text-gray-800">All Bookings</h1>

      {bookings.length === 0 ? (
        <div className="text-center py-12 bg-white rounded-lg">
          <Ticket size={48} className="mx-auto text-gray-400 mb-4" />
          <p className="text-gray-600">No bookings found</p>
        </div>
      ) : (
        <div className="bg-white rounded-lg shadow-md overflow-hidden">
          <div className="overflow-x-auto">
            <table className="w-full">
              <thead className="bg-gray-50 border-b">
                <tr>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Booking ID</th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">User</th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Event</th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Tickets</th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Total</th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Date</th>
                </tr>
              </thead>
              <tbody className="divide-y divide-gray-200">
                {bookings.map((booking) => (
                  <tr key={booking.id} className="hover:bg-gray-50">
                    <td className="px-6 py-4 text-sm text-gray-900">{booking.id}</td>
                    <td className="px-6 py-4 text-sm text-gray-900">{booking.userName}</td>
                    <td className="px-6 py-4 text-sm text-gray-900">{booking.eventName}</td>
                    <td className="px-6 py-4 text-sm text-gray-900">{booking.ticketCount}</td>
                    <td className="px-6 py-4 text-sm font-semibold text-blue-600">${booking.totalPrice}</td>
                    <td className="px-6 py-4 text-sm text-gray-900">
                      {new Date(booking.bookingDate).toLocaleDateString()}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      )}
    </div>
  );
};

const UserProfile: React.FC = () => {
  const { user } = useAuth();
  const [formData, setFormData] = useState({
    name: user?.name || '',
    email: user?.email || ''
  });
  const { token } = useAuth();
  const { showToast } = useToast();
  const { setIsLoading } = useLoader();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsLoading(true);

    try {
      await apiCall('/user/profile', {
        method: 'PUT',
        headers: { Authorization: `Bearer ${token}` },
        body: JSON.stringify(formData)
      });
      showToast('Profile updated successfully!', 'success');
    } catch (error: any) {
      showToast(error.message || 'Update failed', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="space-y-6">
      <h1 className="text-3xl font-bold text-gray-800">My Profile</h1>

      <div className="bg-white rounded-lg shadow-md p-6 max-w-2xl">
        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">Name</label>
            <input
              type="text"
              required
              value={formData.name}
              onChange={(e) => setFormData({ ...formData, name: e.target.value })}
              className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
            />
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">Email</label>
            <input
              type="email"
              required
              value={formData.email}
              onChange={(e) => setFormData({ ...formData, email: e.target.value })}
              className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 outline-none"
            />
          </div>

          <div className="pt-4">
            <button
              type="submit"
              className="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors"
            >
              Update Profile
            </button>
          </div>
        </form>
      </div>
    </div>
  );
};

const Layout: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const { user, logout } = useAuth();
  const [currentRoute, setCurrentRoute] = useState('home');

  const userRoutes = [
    { id: 'home', label: 'Events', icon: Calendar },
    { id: 'bookings', label: 'My Bookings', icon: Ticket },
    { id: 'profile', label: 'Profile', icon: User }
  ];

  const adminRoutes = [
    { id: 'dashboard', label: 'Dashboard', icon: TrendingUp },
    { id: 'events', label: 'Manage Events', icon: Calendar },
    { id: 'bookings', label: 'All Bookings', icon: Ticket }
  ];

  const routes = user?.role === 'admin' ? adminRoutes : userRoutes;

  return (
    <div className="min-h-screen bg-gray-50">
      <nav className="bg-white shadow-md">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="flex justify-between items-center h-16">
            <div className="flex items-center gap-2">
              <Ticket className="text-blue-600" size={32} />
              <span className="text-xl font-bold text-gray-800">TicketBooker</span>
            </div>

            <div className="flex items-center gap-6">
              {routes.map((route) => (
                <button
                  key={route.id}
                  onClick={() => setCurrentRoute(route.id)}
                  className={`flex items-center gap-2 px-4 py-2 rounded-lg transition-colors ${
                    currentRoute === route.id
                      ? 'bg-blue-50 text-blue-600 font-semibold'
                      : 'text-gray-600 hover:bg-gray-50'
                  }`}
                >
                  <route.icon size={20} />
                  {route.label}
                </button>
              ))}

              <button
                onClick={logout}
                className="flex items-center gap-2 px-4 py-2 text-red-600 hover:bg-red-50 rounded-lg transition-colors"
              >
                <LogOut size={20} />
                Logout
              </button>
            </div>
          </div>
        </div>
      </nav>

      <main className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
        {user?.role === 'admin' ? (
          <>
            {currentRoute === 'dashboard' && <AdminDashboard />}
            {currentRoute === 'events' && <AdminEvents />}
            {currentRoute === 'bookings' && <AdminBookings />}
          </>
        ) : (
          <>
            {currentRoute === 'home' && <UserHome />}
            {currentRoute === 'bookings' && <UserBookings />}
            {currentRoute === 'profile' && <UserProfile />}
          </>
        )}
      </main>
    </div>
  );
};

// Main App
const App: React.FC = () => {
  const [mockAuth, setMockAuth] = useState(false);
  const [mockUser, setMockUser] = useState<User | null>(null);

  // Mock login function for demo purposes
  const handleMockLogin = (email: string) => {
    const isAdmin = email.includes('admin');
    setMockUser({
      id: '1',
      email: email,
      name: isAdmin ? 'Admin User' : 'John Doe',
      role: isAdmin ? 'admin' : 'user'
    });
    setMockAuth(true);
  };

  return (
    <ToastProvider>
      <LoaderProvider>
        <AuthProvider>
          {!mockAuth ? (
            <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 flex items-center justify-center p-4">
              <div className="bg-white rounded-2xl shadow-xl p-8 w-full max-w-md">
                <div className="text-center mb-8">
                  <Ticket className="mx-auto text-blue-600 mb-4" size={48} />
                  <h1 className="text-3xl font-bold text-gray-800">Welcome Back</h1>
                  <p className="text-gray-600 mt-2">Sign in to book your tickets</p>
                </div>

                <div className="space-y-4">
                  <button
                    onClick={() => handleMockLogin('user@demo.com')}
                    className="w-full bg-blue-600 text-white py-3 rounded-lg font-semibold hover:bg-blue-700 transition-colors"
                  >
                    Login as User
                  </button>
                  <button
                    onClick={() => handleMockLogin('admin@demo.com')}
                    className="w-full bg-gray-800 text-white py-3 rounded-lg font-semibold hover:bg-gray-900 transition-colors"
                  >
                    Login as Admin
                  </button>
                </div>

                <div className="mt-6 p-4 bg-gray-50 rounded-lg">
                  <p className="text-sm text-gray-600 mb-2">📝 Demo Mode</p>
                  <p className="text-xs text-gray-500">This is a fully functional UI demo. Connect to your backend API by updating the baseURL in the apiCall function.</p>
                </div>
              </div>
            </div>
          ) : (
            <Layout>
              <div />
            </Layout>
          )}
        </AuthProvider>
      </LoaderProvider>
    </ToastProvider>
  );
};

export default App;
