const STORAGE_KEYS = {
  members: 'vcode_members',
  chats: 'vcode_chats',
  reels: 'vcode_reels',
  follows: 'vcode_follows',
  session: 'vcode_session'
};

const state = {
  members: [],
  chats: [],
  reels: [],
  follows: {},
  currentUser: null,
  activeTab: 'home',
  uploadStatus: ''
};

const app = document.getElementById('app');

function loadSavedState() {
  state.members = JSON.parse(localStorage.getItem(STORAGE_KEYS.members) || '[]');
  state.chats = JSON.parse(localStorage.getItem(STORAGE_KEYS.chats) || '[]');
  state.reels = JSON.parse(localStorage.getItem(STORAGE_KEYS.reels) || '[]');
  state.follows = JSON.parse(localStorage.getItem(STORAGE_KEYS.follows) || '{}');
  const session = JSON.parse(localStorage.getItem(STORAGE_KEYS.session) || 'null');

  if (session) {
    state.currentUser = session;
  }
}

function saveMembers() {
  localStorage.setItem(STORAGE_KEYS.members, JSON.stringify(state.members));
}

function saveChats() {
  localStorage.setItem(STORAGE_KEYS.chats, JSON.stringify(state.chats));
}

function saveReels() {
  localStorage.setItem(STORAGE_KEYS.reels, JSON.stringify(state.reels));
}

function saveFollows() {
  localStorage.setItem(STORAGE_KEYS.follows, JSON.stringify(state.follows));
}

function saveSession() {
  if (state.currentUser) {
    localStorage.setItem(STORAGE_KEYS.session, JSON.stringify(state.currentUser));
  } else {
    localStorage.removeItem(STORAGE_KEYS.session);
  }
}

function showToast(message) {
  const toast = document.getElementById('toast');
  if (!toast) return;
  toast.textContent = message;
  toast.classList.add('show');
  clearTimeout(showToast.timer);
  showToast.timer = setTimeout(() => {
    toast.classList.remove('show');
  }, 2200);
}

function getUserRole(username) {
  return username === 'magisenan' ? 'admin' : 'student';
}

function addMemberIfMissing(member) {
  if (!state.members.some((item) => item.username === member.username)) {
    state.members.push(member);
    saveMembers();
  }
}

function getCurrentMember() {
  return state.members.find((member) => member.username === state.currentUser?.username) || null;
}

function getFollowersCount(userId) {
  return state.members.filter((member) => {
    const following = state.follows[member.username] || [];
    return following.includes(userId);
  }).length;
}

function getFollowingList(userId) {
  return state.follows[userId] || [];
}

function getFollowingCount(userId) {
  return getFollowingList(userId).length;
}

function getNotFollowingCount(userId) {
  const myFollowing = getFollowingList(userId);
  return state.members.filter((member) => member.username !== userId && !myFollowing.includes(member.username)).length;
}

function loginView() {
  return `
    <div class="login-shell">
      <div class="login-card">
        <div class="brand">VCODE</div>
        <div class="login-subtitle">Study smarter. Grow together.</div>
        <form id="loginForm" class="form-grid">
          <div class="field-box">
            <label for="username">Username</label>
            <input id="username" name="username" type="text" placeholder="Enter username" required />
          </div>

          <div class="field-box">
            <label for="password">Password</label>
            <input id="password" name="password" type="password" placeholder="Enter password" required />
          </div>

          <div class="field-box">
            <label for="displayName">What is your name?</label>
            <input id="displayName" name="displayName" type="text" placeholder="Enter your name" required />
          </div>

          <button class="primary-button" type="submit">Login</button>
        </form>
        <div id="loginError" class="login-error" aria-live="polite"></div>
      </div>
    </div>
  `;
}

function dashboardView() {
  const currentName = state.currentUser?.name || 'Member';
  const profileLetter = currentName.charAt(0).toUpperCase();
  const followerCount = getFollowersCount(state.currentUser.username);
  const followingCount = getFollowingCount(state.currentUser.username);

  return `
    <div class="dashboard">
      <header class="topbar">
        <div class="logo">VCODE</div>

        <div class="profile-wrap">
          <button class="profile-button" id="profileButton" aria-label="Open profile menu">${profileLetter}</button>
          <div id="profileMenu" class="profile-menu hidden">
            <div class="profile-stat"><span>Followers</span><strong>${followerCount}</strong></div>
            <div class="profile-stat"><span>Following</span><strong>${followingCount}</strong></div>
            <button id="logoutButton" class="logout-button" type="button">Logout</button>
          </div>
        </div>
      </header>

      <main class="content-area">
        <div id="pageContainer" class="page-panel"></div>
      </main>

      <nav class="bottom-nav" aria-label="Main navigation">
        <button class="nav-tab ${state.activeTab === 'home' ? 'active' : ''}" data-tab="home" type="button">Home</button>
        <button class="nav-tab ${state.activeTab === 'chat' ? 'active' : ''}" data-tab="chat" type="button">Chat</button>
        <button class="nav-tab ${state.activeTab === 'newReel' ? 'active' : ''}" data-tab="newReel" type="button">New Reel</button>
        <button class="nav-tab ${state.activeTab === 'followers' ? 'active' : ''}" data-tab="followers" type="button">Followers</button>
        <button class="nav-tab ${state.activeTab === 'exit' ? 'active' : ''}" data-tab="exit" type="button">Exit</button>
      </nav>
    </div>
    <div id="toast" class="toast" aria-live="polite"></div>
  `;
}

function renderHomePage() {
  const reels = [...state.reels].sort((a, b) => b.uploadedAt - a.uploadedAt);

  if (!reels.length) {
    return `
      <div class="empty-state">
        No reels uploaded yet. Only the admin can share the first reel.
      </div>
    `;
  }

  return `
    <div class="feed-list">
      ${reels.map((reel) => {
        const media = reel.fileType.startsWith('video/')
          ? `<video class="reel-media" src="${reel.data}" controls playsinline></video>`
          : `<img class="reel-media" src="${reel.data}" alt="${reel.name}" />`;

        const canDelete = state.currentUser.role === 'admin';

        return `
          <article class="reel-card" data-reel-id="${reel.id}">
            ${media}
            <div class="reel-body">
              <div class="reel-header">
                <div>
                  <div class="reel-name">${reel.name}</div>
                  <div class="reel-meta">Uploaded by ${reel.uploadedByName}</div>
                </div>
                ${canDelete ? `<button class="admin-delete" type="button" data-delete-reel="${reel.id}">Delete</button>` : ''}
              </div>
              <div class="reel-description">${reel.description || 'No description available.'}</div>
            </div>
          </article>
        `;
      }).join('')}
    </div>
  `;
}

function renderChatPage() {
  const messages = [...state.chats].sort((a, b) => a.createdAt - b.createdAt);

  return `
    <div class="chat-panel">
      <div class="chat-box">
        <div class="messages">
          ${messages.length ? messages.map((message) => {
            const canDelete = state.currentUser.role === 'admin' || message.userId === state.currentUser.username;
            return `
              <div class="message-item">
                <div class="message-head">
                  <div class="message-author">${message.userName}</div>
                  <div class="message-time">${new Date(message.createdAt).toLocaleString()}</div>
                </div>
                <div class="message-text">${message.text}</div>
                ${canDelete ? `<div class="message-actions"><button class="delete-button" type="button" data-delete-message="${message.id}">Delete</button></div>` : ''}
              </div>
            `;
          }).join('') : '<div class="empty-state">No messages yet. Start the conversation.</div>'}
        </div>

        <form id="chatForm" class="chat-form">
          <input id="chatInput" type="text" maxlength="400" placeholder="Type your message..." required />
          <button type="submit">Send</button>
        </form>
      </div>
    </div>
  `;
}

function renderNewReelPage() {
  const isAdmin = state.currentUser.role === 'admin';

  return `
    <div class="reel-form">
      ${isAdmin ? `
        <div class="form-card">
          <form id="reelForm">
            <div class="input-group">
              <label for="reelName">Reel Name</label>
              <input id="reelName" name="reelName" type="text" placeholder="Enter reel name" required />
            </div>

            <div class="input-group">
              <label for="reelFile">Choose Image or Video File</label>
              <input id="reelFile" name="reelFile" type="file" accept="image/*,video/*" required />
            </div>

            <div class="input-group">
              <label for="reelDescription">Description</label>
              <textarea id="reelDescription" name="reelDescription" rows="4" placeholder="Write a short description..." required></textarea>
            </div>

            <button class="form-button" type="submit">Upload Reel</button>
            <div id="uploadStatus" class="upload-status">${state.uploadStatus}</div>
          </form>
        </div>
      ` : `
        <div class="empty-state">Only Admin can upload reels.</div>
      `}
    </div>
  `;
}

function renderFollowersPage() {
  const currentUser = state.currentUser;
  const myFollowing = getFollowingList(currentUser.username);

  return `
    <div class="followers-panel">
      <div class="follow-stats">
        <div class="stat-card">
          <div class="stat-label">How many people follow me?</div>
          <div class="stat-value">${getFollowersCount(currentUser.username)}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">How many people am I following?</div>
          <div class="stat-value">${getFollowingCount(currentUser.username)}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">How many members am I not following?</div>
          <div class="stat-value">${getNotFollowingCount(currentUser.username)}</div>
        </div>
      </div>

      <div class="form-card">
        <h3>Registered members</h3>
        <div class="members-grid">
          ${state.members.length ? state.members.map((member) => {
            const isCurrentUser = member.username === currentUser.username;
            const isFollowing = myFollowing.includes(member.username);
            const buttonText = isCurrentUser ? 'You' : (isFollowing ? 'Following' : 'Follow');
            const buttonClass = isCurrentUser ? 'follow-button' : (isFollowing ? 'follow-button following' : 'follow-button');

            return `
              <div class="member-row">
                <div class="member-info">
                  <div class="member-name">${member.name}</div>
                  <div class="member-role">${member.role}</div>
                </div>
                <button class="${buttonClass}" type="button" data-follow-user="${member.username}" ${isCurrentUser ? 'disabled' : ''}>${buttonText}</button>
              </div>
            `;
          }).join('') : '<div class="empty-state">No members yet.</div>'}
        </div>
      </div>
    </div>
  `;
}

function renderPage() {
  const pageContainer = document.getElementById('pageContainer');
  if (!pageContainer) return;

  if (state.activeTab === 'home') pageContainer.innerHTML = renderHomePage();
  if (state.activeTab === 'chat') pageContainer.innerHTML = renderChatPage();
  if (state.activeTab === 'newReel') pageContainer.innerHTML = renderNewReelPage();
  if (state.activeTab === 'followers') pageContainer.innerHTML = renderFollowersPage();
  if (state.activeTab === 'exit') {
    state.currentUser = null;
    saveSession();
    state.activeTab = 'home';
    renderApp();
    return;
  }

  bindGlobalEvents();
}

function bindGlobalEvents() {
  const profileButton = document.getElementById('profileButton');
  const profileMenu = document.getElementById('profileMenu');
  const logoutButton = document.getElementById('logoutButton');
  const navButtons = document.querySelectorAll('.nav-tab');

  if (profileButton && profileMenu) {
    profileButton.addEventListener('click', () => {
      profileMenu.classList.toggle('hidden');
    });
  }

  if (logoutButton) {
    logoutButton.addEventListener('click', () => {
      state.currentUser = null;
      saveSession();
      renderApp();
    });
  }

  navButtons.forEach((button) => {
    button.addEventListener('click', () => {
      const nextTab = button.dataset.tab;
      state.activeTab = nextTab;
      renderApp();
    });
  });

  const deleteReelButtons = document.querySelectorAll('[data-delete-reel]');
  deleteReelButtons.forEach((button) => {
    button.addEventListener('click', () => {
      const reelId = button.dataset.deleteReel;
      if (state.currentUser.role !== 'admin') return;
      state.reels = state.reels.filter((reel) => reel.id !== reelId);
      saveReels();
      renderApp();
    });
  });

  const deleteMessageButtons = document.querySelectorAll('[data-delete-message]');
  deleteMessageButtons.forEach((button) => {
    button.addEventListener('click', () => {
      const messageId = button.dataset.deleteMessage;
      const message = state.chats.find((item) => item.id === messageId);
      if (!message) return;

      const isAdmin = state.currentUser.role === 'admin';
      const isOwner = message.userId === state.currentUser.username;

      if (!isAdmin && !isOwner) {
        return;
      }

      state.chats = state.chats.filter((item) => item.id !== messageId);
      saveChats();
      renderApp();
    });
  });

  const followButtons = document.querySelectorAll('[data-follow-user]');
  followButtons.forEach((button) => {
    button.addEventListener('click', () => {
      const targetUser = button.dataset.followUser;
      if (!targetUser || targetUser === state.currentUser.username) return;

      const myFollowing = getFollowingList(state.currentUser.username);
      const member = state.members.find((item) => item.username === targetUser);
      if (!member) return;

      const exists = myFollowing.includes(targetUser);
      const updated = exists
        ? myFollowing.filter((value) => value !== targetUser)
        : [...myFollowing, targetUser];

      state.follows[state.currentUser.username] = updated;
      saveFollows();
      showToast(exists ? `You unfollowed ${member.name}` : `You are now following ${member.name}`);
      renderApp();
    });
  });

  const chatForm = document.getElementById('chatForm');
  if (chatForm) {
    chatForm.addEventListener('submit', (event) => {
      event.preventDefault();
      const input = document.getElementById('chatInput');
      const text = input.value.trim();
      if (!text) return;

      state.chats.push({
        id: `msg-${Date.now()}-${Math.random().toString(16).slice(2)}`,
        userId: state.currentUser.username,
        userName: state.currentUser.name,
        text,
        createdAt: Date.now()
      });

      saveChats();
      renderApp();
    });
  }

  const reelForm = document.getElementById('reelForm');
  if (reelForm) {
    reelForm.addEventListener('submit', async (event) => {
      event.preventDefault();

      if (state.currentUser.role !== 'admin') {
        state.uploadStatus = 'Only Admin can upload reels.';
        renderApp();
        return;
      }

      const reelNameInput = document.getElementById('reelName');
      const reelDescriptionInput = document.getElementById('reelDescription');
      const reelFileInput = document.getElementById('reelFile');
      const reelName = reelNameInput.value.trim();
      const description = reelDescriptionInput.value.trim();
      const file = reelFileInput.files[0];

      if (!file || !reelName || !description) {
        state.uploadStatus = 'Please complete all reel details.';
        renderApp();
        return;
      }

      const isVideo = file.type.startsWith('video/');
      if (isVideo) {
        const duration = await getVideoDuration(file);
        if (duration > 3600) {
          state.uploadStatus = 'Upload Closed';
          renderApp();
          return;
        }
      }

      const dataUrl = await fileToDataUrl(file);

      const newReel = {
        id: `reel-${Date.now()}-${Math.random().toString(16).slice(2)}`,
        name: reelName,
        description,
        fileType: file.type,
        data: dataUrl,
        uploadedBy: state.currentUser.username,
        uploadedByName: state.currentUser.name,
        uploadedAt: Date.now()
      };

      state.reels.push(newReel);
      saveReels();
      state.activeTab = 'home';
      state.uploadStatus = '';
      renderApp();
    });
  }
}

function fileToDataUrl(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result);
    reader.onerror = () => reject(new Error('Failed to read file'));
    reader.readAsDataURL(file);
  });
}

function getVideoDuration(file) {
  return new Promise((resolve, reject) => {
    const url = URL.createObjectURL(file);
    const video = document.createElement('video');
    video.preload = 'metadata';
    video.src = url;

    video.onloadedmetadata = () => {
      URL.revokeObjectURL(url);
      resolve(video.duration || 0);
    };

    video.onerror = () => {
      URL.revokeObjectURL(url);
      reject(new Error('Video metadata unavailable'));
    };
  });
}

function handleLogin(event) {
  event.preventDefault();
  const form = event.currentTarget;
  const username = form.username.value.trim();
  const password = form.password.value.trim();
  const enteredName = form.displayName.value.trim();

  if ((username === 'magisenan' && password === 'magisenan9') || (username === 'vcode kit students' && password === 'vcode kit students')) {
    const member = {
      username,
      name: enteredName || username,
      role: getUserRole(username),
      memberSince: Date.now()
    };

    addMemberIfMissing(member);
    state.currentUser = member;
    saveSession();
    state.activeTab = 'home';
    renderApp();
    return;
  }

  const errorBox = document.getElementById('loginError');
  if (errorBox) {
    errorBox.textContent = 'Wrong Username or Password';
  }
}

function renderApp() {
  if (!state.currentUser) {
    app.innerHTML = loginView();
    const form = document.getElementById('loginForm');
    if (form) {
      form.addEventListener('submit', handleLogin);
    }
  } else {
    app.innerHTML = dashboardView();
    renderPage();
  }
}

loadSavedState();
renderApp();
