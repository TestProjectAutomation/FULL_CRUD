<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'

interface User {
  id: number
  name: string
  email: string
  password: string
}

// 🔗 Django API base URL
const API_URL = 'http://127.0.0.1:8000/api/users/'

// 🔸 Reactive variables
const users = ref<User[]>([])
const newUser = ref<User>({ id: 0, name: '', email: '', password: '' })
const isEditing = ref(false)
const editingId = ref<number | null>(null)
const message = ref('')
const showMessage = ref(false)
const messageType = ref<'success' | 'error' | 'info'>('success')
const confirmVisible = ref(false)
const confirmUserId = ref<number | null>(null)
const isLoading = ref(false)
const searchQuery = ref('')
const showPassword = ref(false)
const passwordStrength = ref(0)

// 🎨 Background images
const images = ['/bg1.jpg', '/bg4.jpg', '/bg2.png', '/bg3.jpg']
const currentImageIndex = ref(0)
const currentImage = computed(() => images[currentImageIndex.value])

// 📊 Filtered users based on search
const filteredUsers = computed(() => {
  if (!searchQuery.value) return users.value
  return users.value.filter(user => 
    user.name.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
    user.email.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
})

// 💪 Password strength checker
const checkPasswordStrength = (password: string) => {
  let strength = 0
  if (password.length >= 6) strength++
  if (password.length >= 10) strength++
  if (/[A-Z]/.test(password)) strength++
  if (/[0-9]/.test(password)) strength++
  if (/[^A-Za-z0-9]/.test(password)) strength++
  return Math.min(strength, 4)
}

const updatePasswordStrength = () => {
  passwordStrength.value = checkPasswordStrength(newUser.value.password)
}

const getStrengthText = () => {
  if (passwordStrength.value === 0) return 'ضعيفة جداً'
  if (passwordStrength.value === 1) return 'ضعيفة'
  if (passwordStrength.value === 2) return 'متوسطة'
  if (passwordStrength.value === 3) return 'جيدة'
  return 'قوية جداً'
}

const getStrengthColor = () => {
  if (passwordStrength.value <= 1) return '#ef4444'
  if (passwordStrength.value === 2) return '#f59e0b'
  if (passwordStrength.value === 3) return '#3b82f6'
  return '#22c55e'
}

// 🌈 Auto change background
onMounted(() => {
  setInterval(() => {
    currentImageIndex.value = (currentImageIndex.value + 1) % images.length
  }, 15000)
  fetchUsers()
})

// 🎯 Popup message
const showPopup = (msg: string, type: 'success' | 'error' | 'info' = 'success') => {
  message.value = msg
  messageType.value = type
  showMessage.value = true
  setTimeout(() => (showMessage.value = false), 2500)
}

// 🧩 Fetch all users
const fetchUsers = async () => {
  isLoading.value = true
  try {
    const res = await fetch(API_URL)
    if (!res.ok) throw new Error('Failed to fetch users')
    users.value = await res.json()
  } catch (err: any) {
    showPopup(`⚠️ ${err.message}`, 'error')
  } finally {
    setTimeout(() => (isLoading.value = false), 500)
  }
}

// ➕ Add user
const addUser = async () => {
  if (!newUser.value.name || !newUser.value.email || !newUser.value.password) {
    showPopup('Please fill all fields', 'error')
    return
  }
  
  if (!isValidEmail(newUser.value.email)) {
    showPopup('Please enter a valid email', 'error')
    return
  }
  
  if (newUser.value.password.length < 6) {
    showPopup('Password must be at least 6 characters', 'error')
    return
  }
  
  isLoading.value = true
  try {
    const res = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: newUser.value.name,
        email: newUser.value.email,
        password: newUser.value.password,
      }),
    })

    if (!res.ok) throw new Error('Failed to add user')

    await fetchUsers()
    resetForm()
    showPopup('✨ User added successfully!', 'success')
  } catch (err: any) {
    showPopup(`⚠️ ${err.message}`, 'error')
  } finally {
    isLoading.value = false
  }
}

// ✏️ Edit user
const editUser = (user: User) => {
  isEditing.value = true
  editingId.value = user.id
  newUser.value = { ...user, password: '' } // Don't show password when editing
  // Smooth scroll to form
  document.querySelector('.form-container')?.scrollIntoView({ 
    behavior: 'smooth', 
    block: 'center' 
  })
}

// ♻️ Update user
const updateUser = async () => {
  if (!editingId.value) return
  
  isLoading.value = true
  try {
    const updateData: any = {
      name: newUser.value.name,
      email: newUser.value.email,
    }
    
    // Only send password if it's provided
    if (newUser.value.password) {
      updateData.password = newUser.value.password
    }
    
    const res = await fetch(`${API_URL}${editingId.value}/`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(updateData),
    })

    if (!res.ok) throw new Error('Failed to update user')

    await fetchUsers()
    resetForm()
    showPopup('🔄 User updated successfully!', 'success')
  } catch (err: any) {
    showPopup(`⚠️ ${err.message}`, 'error')
  } finally {
    isLoading.value = false
  }
}

// 🔄 Reset form
const resetForm = () => {
  isEditing.value = false
  editingId.value = null
  newUser.value = { id: 0, name: '', email: '', password: '' }
  passwordStrength.value = 0
  showPassword.value = false
}

// ❌ Cancel editing
const cancelEdit = () => {
  if (isEditing.value) {
    resetForm()
    showPopup('Edit cancelled', 'info')
  }
}

// 🗑️ Delete user
const deleteUser = async (id: number) => {
  isLoading.value = true
  try {
    const res = await fetch(`${API_URL}${id}/`, { method: 'DELETE' })
    if (!res.ok) throw new Error('Failed to delete user')
    await fetchUsers()
    showPopup('🗑️ User deleted successfully!', 'success')
  } catch (err: any) {
    showPopup(`⚠️ ${err.message}`, 'error')
  } finally {
    isLoading.value = false
  }
}

// ✅ Email validation
const isValidEmail = (email: string) => {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
}

// 🔓 Confirmation modal
const openConfirm = (id: number) => {
  confirmUserId.value = id
  confirmVisible.value = true
}

const confirmDelete = async () => {
  if (confirmUserId.value !== null) {
    await deleteUser(confirmUserId.value)
    confirmVisible.value = false
    confirmUserId.value = null
  }
}

const cancelDelete = () => {
  confirmVisible.value = false
  confirmUserId.value = null
}

// 👁️ Toggle password visibility
const togglePasswordVisibility = () => {
  showPassword.value = !showPassword.value
}
</script>

<template>
  <div class="background" :style="{ backgroundImage: `url(${currentImage})` }">
    <div class="overlay">
      <!-- Modern Header -->
      <div class="header">
        <h1 class="title">
          <span class="title-icon">✨</span>
          User Management
          <span class="title-badge">CRUD</span>
        </h1>
        <p class="subtitle">Manage your users efficiently with modern interface</p>
      </div>

      <!-- Loading Spinner -->
      <transition name="fade">
        <div v-if="isLoading" class="loading-overlay">
          <div class="spinner"></div>
        </div>
      </transition>

      <!-- Popup Notification -->
      <transition name="slide-down">
        <div
          v-if="showMessage"
          class="notification"
          :class="`notification-${messageType}`"
        >
          <span class="notification-icon">
            {{ messageType === 'success' ? '✅' : messageType === 'error' ? '⚠️' : 'ℹ️' }}
          </span>
          <span>{{ message }}</span>
        </div>
      </transition>

      <!-- Delete Confirmation Modal -->
      <transition name="modal">
        <div v-if="confirmVisible" class="modal-overlay" @click.self="cancelDelete">
          <div class="modal-container">
            <div class="modal-icon">🗑️</div>
            <h3>Confirm Deletion</h3>
            <p>Are you sure you want to delete this user? This action cannot be undone.</p>
            <div class="modal-actions">
              <button class="btn btn-secondary" @click="cancelDelete">Cancel</button>
              <button class="btn btn-danger" @click="confirmDelete">Delete</button>
            </div>
          </div>
        </div>
      </transition>

      <!-- Stats Cards -->
      <div class="stats-container">
        <div class="stat-card">
          <div class="stat-icon">👥</div>
          <div class="stat-info">
            <h3>{{ users.length }}</h3>
            <p>Total Users</p>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon">📊</div>
          <div class="stat-info">
            <h3>{{ filteredUsers.length }}</h3>
            <p>Filtered Users</p>
          </div>
        </div>
      </div>

      <!-- Search Bar -->
      <div class="search-container">
        <div class="search-box">
          <span class="search-icon">🔍</span>
          <input
            v-model="searchQuery"
            type="text"
            placeholder="Search by name or email..."
            class="search-input"
          />
          <button v-if="searchQuery" class="search-clear" @click="searchQuery = ''">✕</button>
        </div>
      </div>

      <!-- Modern Table -->
      <div class="table-wrapper">
        <div class="table-container">
          <table class="user-table">
            <thead>
              <tr>
                <th>#</th>
                <th>Name</th>
                <th>Email</th>
                <th>Actions</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="filteredUsers.length === 0">
                <td colspan="4" class="empty-state">
                  <div class="empty-icon">📭</div>
                  <p>No users found</p>
                </td>
              </tr>
              <tr v-for="(user, index) in filteredUsers" :key="user.id">
                <td data-label="#">#{{ index + 1 }}</td>
                <td data-label="Name">
                  <div class="user-name">
                    <div class="user-avatar">{{ user.name.charAt(0).toUpperCase() }}</div>
                    {{ user.name }}
                  </div>
                </td>
                <td data-label="Email">{{ user.email }}</td>
                <td data-label="Actions">
                  <div class="action-buttons">
                    <button @click="editUser(user)" class="action-btn edit-btn" title="Edit">
                      ✏️
                    </button>
                    <button @click="openConfirm(user.id)" class="action-btn delete-btn" title="Delete">
                      🗑️
                    </button>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Modern Form with Password -->
      <div class="form-wrapper">
        <div class="form-container">
          <div class="form-header">
            <h2>{{ isEditing ? '✏️ Edit User' : '➕ Add New User' }}</h2>
            <div class="form-decoration"></div>
          </div>
          
          <div class="form-body">
            <div class="input-group">
              <label>Full Name</label>
              <div class="input-icon">
                <span>👤</span>
                <input
                  v-model="newUser.name"
                  type="text"
                  placeholder="Enter full name"
                  @keyup.enter="isEditing ? updateUser() : addUser()"
                />
              </div>
            </div>
            
            <div class="input-group">
              <label>Email Address</label>
              <div class="input-icon">
                <span>📧</span>
                <input
                  v-model="newUser.email"
                  type="email"
                  placeholder="Enter email address"
                  @keyup.enter="isEditing ? updateUser() : addUser()"
                />
              </div>
            </div>

            <!-- Password Field -->
            <div class="input-group">
              <label>Password 
                <span v-if="isEditing" class="optional-badge">(Optional - leave blank to keep current)</span>
              </label>
              <div class="input-icon password-field">
                <span>🔒</span>
                <input
                  v-model="newUser.password"
                  :type="showPassword ? 'text' : 'password'"
                  placeholder="Enter password"
                  @input="updatePasswordStrength"
                  @keyup.enter="isEditing ? updateUser() : addUser()"
                />
                <button type="button" class="password-toggle" @click="togglePasswordVisibility">
                  {{ showPassword ? '🙈' : '👁️' }}
                </button>
              </div>
              
              <!-- Password Strength Indicator -->
              <div v-if="newUser.password" class="password-strength">
                <div class="strength-bar">
                  <div 
                    class="strength-fill" 
                    :style="{ 
                      width: `${(passwordStrength / 4) * 100}%`,
                      backgroundColor: getStrengthColor()
                    }"
                  ></div>
                </div>
                <div class="strength-text" :style="{ color: getStrengthColor() }">
                  {{ getStrengthText() }}
                </div>
              </div>
              
              <div class="password-hint">
                <small>🔐 Must be at least 6 characters with letters & numbers</small>
              </div>
            </div>

            <div class="form-actions">
              <button v-if="!isEditing" @click="addUser" class="btn btn-primary" :disabled="isLoading">
                <span>✨</span> Add User
              </button>
              
              <template v-else>
                <button @click="updateUser" class="btn btn-primary" :disabled="isLoading">
                  <span>🔄</span> Update
                </button>
                <button @click="cancelEdit" class="btn btn-secondary">
                  <span>❌</span> Cancel
                </button>
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
  /* إضافة التنسيقات الجديدة لحقل كلمة المرور */
  .password-field {
    position: relative;
  }

  .password-toggle {
    position: absolute;
    right: 15px;
    background: none;
    border: none;
    cursor: pointer;
    font-size: 20px;
    padding: 0;
    opacity: 0.7;
    transition: opacity 0.2s;
  }

  .password-toggle:hover {
    opacity: 1;
  }

  .password-strength {
    margin-top: 8px;
  }

  .strength-bar {
    height: 4px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 2px;
    overflow: hidden;
    margin-bottom: 5px;
  }

  .strength-fill {
    height: 100%;
    transition: width 0.3s ease, background-color 0.3s ease;
  }

  .strength-text {
    font-size: 12px;
    font-weight: 500;
    transition: color 0.3s ease;
  }

  .password-hint {
    margin-top: 5px;
  }

  .password-hint small {
    color: rgba(255, 255, 255, 0.5);
    font-size: 11px;
  }

  .optional-badge {
    font-size: 11px;
    font-weight: normal;
    color: rgba(255, 255, 255, 0.6);
    background: rgba(255, 255, 255, 0.1);
    padding: 2px 8px;
    border-radius: 12px;
    margin-left: 8px;
  }

  /* باقي التنسيقات كما هي */
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  /* Background */
  .background {
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    overflow: auto;
    transition: background-image 1.5s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .overlay {
    min-height: 100vh;
    padding: 40px 20px;
    background: linear-gradient(135deg, rgba(0, 0, 0, 0.7), rgba(0, 0, 0, 0.5));
    backdrop-filter: blur(2px);
  }

  /* Header */
  .header {
    text-align: center;
    margin-bottom: 40px;
    animation: fadeInUp 0.8s ease;
  }

  .title {
    font-size: 48px;
    font-weight: 800;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
    display: inline-flex;
    align-items: center;
    gap: 15px;
    flex-wrap: wrap;
    justify-content: center;
  }

  .title-icon {
    font-size: 48px;
    animation: rotate 3s infinite linear;
  }

  .title-badge {
    font-size: 14px;
    background: linear-gradient(135deg, #667eea, #764ba2);
    color: white;
    padding: 5px 12px;
    border-radius: 20px;
    letter-spacing: 1px;
  }

  .subtitle {
    color: rgba(255, 255, 255, 0.8);
    font-size: 18px;
    margin-top: 10px;
  }

  /* Stats Cards */
  .stats-container {
    display: flex;
    justify-content: center;
    gap: 20px;
    margin-bottom: 30px;
    flex-wrap: wrap;
  }

  .stat-card {
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    border-radius: 20px;
    padding: 20px 30px;
    display: flex;
    align-items: center;
    gap: 15px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    transition: all 0.3s ease;
    animation: fadeInUp 0.8s ease 0.1s both;
  }

  .stat-card:hover {
    transform: translateY(-5px);
    background: rgba(255, 255, 255, 0.15);
    border-color: rgba(255, 255, 255, 0.4);
  }

  .stat-icon {
    font-size: 40px;
  }

  .stat-info h3 {
    font-size: 32px;
    color: white;
    font-weight: 700;
  }

  .stat-info p {
    color: rgba(255, 255, 255, 0.7);
    font-size: 14px;
  }

  /* Search */
  .search-container {
    display: flex;
    justify-content: center;
    margin-bottom: 30px;
  }

  .search-box {
    position: relative;
    width: 100%;
    max-width: 400px;
  }

  .search-icon {
    position: absolute;
    left: 15px;
    top: 50%;
    transform: translateY(-50%);
    font-size: 18px;
    opacity: 0.7;
  }

  .search-input {
    width: 100%;
    padding: 14px 40px 14px 45px;
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 50px;
    color: white;
    font-size: 16px;
    transition: all 0.3s ease;
  }

  .search-input:focus {
    outline: none;
    background: rgba(255, 255, 255, 0.15);
    border-color: #667eea;
    box-shadow: 0 0 20px rgba(102, 126, 234, 0.3);
  }

  .search-input::placeholder {
    color: rgba(255, 255, 255, 0.5);
  }

  .search-clear {
    position: absolute;
    right: 15px;
    top: 50%;
    transform: translateY(-50%);
    background: rgba(255, 255, 255, 0.2);
    border: none;
    color: white;
    width: 24px;
    height: 24px;
    border-radius: 50%;
    cursor: pointer;
    transition: all 0.2s;
  }

  .search-clear:hover {
    background: rgba(255, 255, 255, 0.4);
  }

  /* Table */
  .table-wrapper {
    display: flex;
    justify-content: center;
    margin-bottom: 40px;
  }

  .table-container {
    width: 90%;
    max-width: 1200px;
    background: rgba(255, 255, 255, 0.05);
    backdrop-filter: blur(10px);
    border-radius: 20px;
    padding: 20px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    animation: fadeInUp 0.8s ease 0.2s both;
  }

  .user-table {
    width: 100%;
    border-collapse: collapse;
    color: white;
  }

  .user-table thead th {
    padding: 15px;
    text-align: left;
    font-weight: 600;
    font-size: 14px;
    text-transform: uppercase;
    letter-spacing: 1px;
    color: rgba(255, 255, 255, 0.7);
    border-bottom: 2px solid rgba(255, 255, 255, 0.1);
  }

  .user-table tbody tr {
    transition: all 0.3s ease;
    border-bottom: 1px solid rgba(255, 255, 255, 0.05);
  }

  .user-table tbody tr:hover {
    background: rgba(255, 255, 255, 0.1);
    transform: translateX(5px);
  }

  .user-table td {
    padding: 15px;
    vertical-align: middle;
  }

  .user-name {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .user-avatar {
    width: 35px;
    height: 35px;
    background: linear-gradient(135deg, #667eea, #764ba2);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: bold;
    font-size: 16px;
  }

  /* Action Buttons */
  .action-buttons {
    display: flex;
    gap: 10px;
  }

  .action-btn {
    width: 35px;
    height: 35px;
    border: none;
    border-radius: 10px;
    cursor: pointer;
    font-size: 18px;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .edit-btn {
    background: linear-gradient(135deg, #f59e0b, #f97316);
    color: white;
  }

  .edit-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 5px 15px rgba(245, 158, 11, 0.4);
  }

  .delete-btn {
    background: linear-gradient(135deg, #ef4444, #dc2626);
    color: white;
  }

  .delete-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 5px 15px rgba(239, 68, 68, 0.4);
  }

  /* Empty State */
  .empty-state {
    text-align: center;
    padding: 60px 20px !important;
  }

  .empty-icon {
    font-size: 64px;
    margin-bottom: 15px;
    opacity: 0.5;
  }

  .empty-state p {
    font-size: 18px;
    color: rgba(255, 255, 255, 0.5);
  }

  /* Form */
  .form-wrapper {
    display: flex;
    justify-content: center;
  }

  .form-container {
    width: 100%;
    max-width: 500px;
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(20px);
    border-radius: 25px;
    overflow: hidden;
    border: 1px solid rgba(255, 255, 255, 0.2);
    animation: fadeInUp 0.8s ease 0.3s both;
  }

  .form-header {
    background: linear-gradient(135deg, rgba(102, 126, 234, 0.2), rgba(118, 75, 162, 0.2));
    padding: 25px;
    text-align: center;
    position: relative;
  }

  .form-header h2 {
    color: white;
    font-size: 24px;
    margin: 0;
  }

  .form-decoration {
    position: absolute;
    bottom: -2px;
    left: 0;
    right: 0;
    height: 3px;
    background: linear-gradient(90deg, #667eea, #764ba2, #667eea);
    background-size: 200%;
    animation: gradient 3s ease infinite;
  }

  .form-body {
    padding: 30px;
  }

  .input-group {
    margin-bottom: 25px;
  }

  .input-group label {
    display: block;
    color: white;
    margin-bottom: 8px;
    font-size: 14px;
    font-weight: 500;
  }

  .input-icon {
    position: relative;
    display: flex;
    align-items: center;
  }

  .input-icon span {
    position: absolute;
    left: 15px;
    font-size: 18px;
    opacity: 0.7;
  }

  .input-icon input {
    width: 100%;
    padding: 14px 15px 14px 45px;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 12px;
    color: white;
    font-size: 16px;
    transition: all 0.3s ease;
  }

  .input-icon input:focus {
    outline: none;
    background: rgba(255, 255, 255, 0.15);
    border-color: #667eea;
    box-shadow: 0 0 15px rgba(102, 126, 234, 0.3);
  }

  .input-icon input::placeholder {
    color: rgba(255, 255, 255, 0.4);
  }

  .form-actions {
    display: flex;
    gap: 15px;
    margin-top: 30px;
  }

  .btn {
    flex: 1;
    padding: 14px;
    border: none;
    border-radius: 12px;
    font-size: 16px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
  }

  .btn-primary {
    background: linear-gradient(135deg, #667eea, #764ba2);
    color: white;
  }

  .btn-primary:hover:not(:disabled) {
    transform: translateY(-2px);
    box-shadow: 0 8px 20px rgba(102, 126, 234, 0.4);
  }

  .btn-secondary {
    background: rgba(255, 255, 255, 0.2);
    color: white;
  }

  .btn-secondary:hover {
    background: rgba(255, 255, 255, 0.3);
    transform: translateY(-2px);
  }

  .btn-danger {
    background: linear-gradient(135deg, #ef4444, #dc2626);
    color: white;
  }

  .btn-danger:hover {
    transform: translateY(-2px);
    box-shadow: 0 8px 20px rgba(239, 68, 68, 0.4);
  }

  .btn:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  /* Modal */
  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.8);
    backdrop-filter: blur(8px);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }

  .modal-container {
    background: rgba(255, 255, 255, 0.95);
    border-radius: 25px;
    padding: 40px;
    max-width: 400px;
    width: 90%;
    text-align: center;
    animation: modalSlideIn 0.3s ease;
  }

  .modal-icon {
    font-size: 64px;
    margin-bottom: 20px;
  }

  .modal-container h3 {
    font-size: 24px;
    margin-bottom: 10px;
    color: #333;
  }

  .modal-container p {
    color: #666;
    margin-bottom: 25px;
  }

  .modal-actions {
    display: flex;
    gap: 15px;
  }

  /* Notification */
  .notification {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 15px 25px;
    border-radius: 12px;
    display: flex;
    align-items: center;
    gap: 12px;
    font-weight: 500;
    z-index: 2000;
    animation: slideInRight 0.3s ease;
    backdrop-filter: blur(10px);
  }

  .notification-success {
    background: rgba(34, 197, 94, 0.9);
    color: white;
    border-left: 4px solid #22c55e;
  }

  .notification-error {
    background: rgba(239, 68, 68, 0.9);
    color: white;
    border-left: 4px solid #ef4444;
  }

  .notification-info {
    background: rgba(59, 130, 246, 0.9);
    color: white;
    border-left: 4px solid #3b82f6;
  }

  /* Loading Spinner */
  .loading-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    backdrop-filter: blur(4px);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 3000;
  }

  .spinner {
    width: 50px;
    height: 50px;
    border: 3px solid rgba(255, 255, 255, 0.3);
    border-top-color: #667eea;
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
  }

  /* Animations */
  @keyframes fadeInUp {
    from {
      opacity: 0;
      transform: translateY(30px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  @keyframes slideInRight {
    from {
      opacity: 0;
      transform: translateX(100px);
    }
    to {
      opacity: 1;
      transform: translateX(0);
    }
  }

  @keyframes slideDown {
    from {
      opacity: 0;
      transform: translateY(-20px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  @keyframes modalSlideIn {
    from {
      opacity: 0;
      transform: scale(0.9);
    }
    to {
      opacity: 1;
      transform: scale(1);
    }
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  @keyframes rotate {
    from {
      transform: rotate(0deg);
    }
    to {
      transform: rotate(360deg);
    }
  }

  @keyframes gradient {
    0% {
      background-position: 0% 50%;
    }
    50% {
      background-position: 100% 50%;
    }
    100% {
      background-position: 0% 50%;
    }
  }

  /* Transitions */
  .fade-enter-active,
  .fade-leave-active {
    transition: opacity 0.3s ease;
  }

  .fade-enter-from,
  .fade-leave-to {
    opacity: 0;
  }

  .slide-down-enter-active,
  .slide-down-leave-active {
    transition: all 0.3s ease;
  }

  .slide-down-enter-from,
  .slide-down-leave-to {
    opacity: 0;
    transform: translateY(-20px);
  }

  .modal-enter-active,
  .modal-leave-active {
    transition: all 0.3s ease;
  }

  .modal-enter-from,
  .modal-leave-to {
    opacity: 0;
    transform: scale(0.9);
  }

  /* Responsive */
  @media (max-width: 768px) {
    .title {
      font-size: 32px;
    }
    
    .title-icon {
      font-size: 32px;
    }
    
    .subtitle {
      font-size: 14px;
    }
    
    .stat-card {
      padding: 15px 20px;
    }
    
    .stat-info h3 {
      font-size: 24px;
    }
    
    .table-container {
      padding: 10px;
      overflow-x: auto;
    }
    
    .user-table {
      min-width: 600px;
    }
    
    .form-container {
      margin: 0 20px;
    }
    
    .modal-container {
      padding: 25px;
      margin: 20px;
    }
    
    .notification {
      top: auto;
      bottom: 20px;
      right: 20px;
      left: 20px;
    }
  }

  @media (max-width: 480px) {
    .overlay {
      padding: 20px 15px;
    }
    
    .stats-container {
      gap: 10px;
    }
    
    .stat-card {
      padding: 12px 18px;
    }
    
    .form-body {
      padding: 20px;
    }
    
    .btn {
      padding: 12px;
      font-size: 14px;
    }
    
    .optional-badge {
      display: block;
      margin-top: 5px;
      margin-left: 0;
    }
  }
</style>