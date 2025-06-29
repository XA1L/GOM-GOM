<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Mini Sosyal Medya Prototipi</title>
<style>
  body { font-family: Arial, sans-serif; background: #f0f2f5; margin:0; padding:0; }
  header { background: #1877f2; color: white; padding: 15px; text-align: center; font-size: 24px; }
  #auth, #app { max-width: 700px; margin: 20px auto; background: white; border-radius: 8px; padding: 15px; box-shadow: 0 0 10px #ccc;}
  input, button, textarea, select { width: 100%; padding: 8px; margin: 8px 0; box-sizing: border-box; font-size: 16px; }
  button { background: #1877f2; color: white; border: none; border-radius: 4px; cursor: pointer; }
  button:hover { background: #155db2; }
  nav { margin-bottom: 15px; }
  nav button { width: auto; padding: 8px 16px; margin-right: 8px; }
  .profile-pic { width: 70px; height: 70px; border-radius: 50%; object-fit: cover; border: 2px solid #1877f2; }
  .post { border: 1px solid #ddd; margin-bottom: 15px; border-radius: 6px; padding: 10px; background: #fff; }
  .post-header { display: flex; align-items: center; margin-bottom: 10px; }
  .post-header img { margin-right: 10px; }
  .post-content img { max-width: 100%; margin-top: 10px; border-radius: 6px; }
  .btn-like, .btn-comment { cursor: pointer; color: #555; margin-right: 10px; }
  .btn-like.liked { color: #e0245e; font-weight: bold; }
  .comments { margin-top: 10px; padding-left: 20px; }
  .comment { margin-bottom: 8px; border-left: 2px solid #ddd; padding-left: 8px; }
  .reply { margin-left: 15px; border-left: 2px solid #eee; padding-left: 8px; font-size: 14px; color: #555; }
  .message-list { max-height: 300px; overflow-y: auto; border: 1px solid #ddd; padding: 10px; background: #fff; margin-bottom: 10px; border-radius: 6px; }
  .message { margin-bottom: 8px; }
  .message.self { text-align: right; }
  small { color: #999; }
  .verified { color: #3897f0; font-weight: bold; margin-left: 6px; }
  .error { color: red; }
  .success { color: green; }
</style>
</head>
<body>

<header>GOM GOM</header>

<div id="auth">
  <h2>Kayıt / Giriş</h2>
  <input type="text" id="username" placeholder="Kullanıcı adı (benzersiz)" />
  <input type="password" id="password" placeholder="Şifre (en az 8 karakter, harf ve sayı)" />
  <button onclick="register()">Kayıt Ol</button>
  <button onclick="login()">Giriş Yap</button>
  <p id="auth-msg" class="error"></p>
</div>

<div id="app" style="display:none;">
  <nav>
    <button onclick="showFeed('following')">Takip Ettiklerim</button>
    <button onclick="showFeed('global')">Global Akış</button>
    <button onclick="showProfile(currentUser)">Profilim</button>
    <button onclick="showUsers()">Kullanıcılar</button>
    <button onclick="logout()">Çıkış</button>
  </nav>

  <div id="feed"></div>

  <div id="profile" style="display:none;">
    <h3>Profil: <span id="profileName"></span></h3>
    <img id="profilePic" class="profile-pic" src="https://via.placeholder.com/70?text=Profil" alt="Profil Foto" />
    <input type="file" id="profilePicInput" accept="image/*" onchange="changeProfilePic(event)" />
    <textarea id="bio" placeholder="Biyografi" rows="3"></textarea>
    <input type="password" id="newPassword" placeholder="Yeni Şifre (değiştirmek için)" />
    <button onclick="saveProfile()">Profili Kaydet</button>
    <button id="verifyBtn" style="display:none;" onclick="">Mavi Tik Ver</button>
  </div>

  <div id="users" style="display:none;">
    <h3>Kullanıcılar</h3>
    <div id="userList"></div>
  </div>

  <div id="messages" style="display:none;">
    <h3>Mesajlar</h3>
    <select id="chatUserSelect" onchange="loadChat(this.value)">
      <option value="">Kullanıcı Seç</option>
    </select>
    <div class="message-list" id="messageList"></div>
    <textarea id="messageInput" placeholder="Mesaj yaz..." rows="3"></textarea>
    <button onclick="sendMessage()">Gönder</button>
  </div>

</div>

<script>
// ---------- Veri ve Durum ----------

let currentUser = null;
let currentFeed = 'following'; // 'global' veya 'following'
let currentChatUser = null;

function loadUsers() {
  return JSON.parse(localStorage.getItem('msm_users') || '{}');
}
function saveUsers(users) {
  localStorage.setItem('msm_users', JSON.stringify(users));
}
function loadPosts() {
  return JSON.parse(localStorage.getItem('msm_posts') || '[]');
}
function savePosts(posts) {
  localStorage.setItem('msm_posts', JSON.stringify(posts));
}

// ---------- Başlangıç Admin ve Test Kullanıcısı ----------

function init() {
  let users = loadUsers();
  if(Object.keys(users).length === 0) {
    users = {
      admin: {
        password: 'sifre123',
        bio: '',
        profilePic: '',
        followers: [],
        following: [],
        isVerified: true,
        canVerifyOthers: true,
        messages: {}
      },
      sa: {
        password: '12345678',
        bio: 'Ben birisiyim ',
        profilePic: '',
        followers: [],
        following: [],
        isVerified: false,
        canVerifyOthers: false,
        messages: {}
      }
    };
    saveUsers(users);
  }
  if(!localStorage.getItem('msm_posts')) {
    savePosts([]);
  }
}

init();

// ---------- Kayıt ve Giriş ----------

function validPassword(pwd) {
  return pwd.length >=8 && /[a-zA-Z]/.test(pwd) && /\d/.test(pwd);
}

function register() {
  const u = document.getElementById('username').value.trim();
  const p = document.getElementById('password').value;
  const msg = document.getElementById('auth-msg');
  msg.textContent = '';
  if(!u || !p) { msg.textContent = 'Kullanıcı adı ve şifre gerekli!'; return; }
  if(!validPassword(p)) { msg.textContent = 'Şifre en az 8 karakter, harf ve sayı içermeli!'; return; }
  let users = loadUsers();
  if(users[u]) { msg.textContent = 'Bu kullanıcı adı alınmış!'; return; }
  users[u] = {
    password: p,
    bio: '',
    profilePic: '',
    followers: [],
    following: [],
    isVerified: false,
    canVerifyOthers: false,
    messages: {}
  };
  saveUsers(users);
  msg.style.color = 'green';
  msg.textContent = 'Kayıt başarılı! Giriş yapabilirsiniz.';
}

function login() {
  const u = document.getElementById('username').value.trim();
  const p = document.getElementById('password').value;
  const msg = document.getElementById('auth-msg');
  msg.style.color = 'red';
  msg.textContent = '';
  let users = loadUsers();
  if(!users[u] || users[u].password !== p) {
    msg.textContent = 'Kullanıcı adı veya şifre yanlış!';
    return;
  }
  currentUser = u;
  document.getElementById('auth').style.display = 'none';
  document.getElementById('app').style.display = 'block';
  showFeed('following');
  loadUserSelect();
}

// ---------- Çıkış ----------

function logout() {
  currentUser = null;
  currentFeed = 'following';
  currentChatUser = null;
  document.getElementById('auth').style.display = 'block';
  document.getElementById('app').style.display = 'none';
  document.getElementById('auth-msg').textContent = '';
  document.getElementById('username').value = '';
  document.getElementById('password').value = '';
  document.getElementById('feed').style.display = 'block';
  document.getElementById('profile').style.display = 'none';
  document.getElementById('users').style.display = 'none';
  document.getElementById('messages').style.display = 'none';
}

// ---------- Profil ----------

function showProfile(username) {
  if(!username) return;
  const users = loadUsers();
  if(!users[username]) return alert('Kullanıcı bulunamadı');
  const user = users[username];

  document.getElementById('feed').style.display = 'none';
  document.getElementById('profile').style.display = 'block';
  document.getElementById('users').style.display = 'none';
  document.getElementById('messages').style.display = 'none';

  document.getElementById('profileName').innerHTML = username + (user.isVerified ? '<span class="verified"> ✔️</span>' : '');
  document.getElementById('bio').value = user.bio || '';
  document.getElementById('profilePic').src = user.profilePic || 'https://via.placeholder.com/70?text=Profil';

  if(username === currentUser) {
    document.getElementById('profilePicInput').style.display = 'block';
    document.getElementById('bio').disabled = false;
    document.getElementById('newPassword').style.display = 'block';
    document.getElementById('verifyBtn').style.display = 'none';
  } else {
    document.getElementById('profilePicInput').style.display = 'none';
    document.getElementById('bio').disabled = true;
    document.getElementById('newPassword').style.display = 'none';
    const verifyBtn = document.getElementById('verifyBtn');
    if(currentUser === 'admin' && !user.isVerified) {
      verifyBtn.style.display = 'inline-block';
      verifyBtn.onclick = () => verifyUser(username);
    } else {
      verifyBtn.style.display = 'none';
    }
  }
}

function changeProfilePic(event) {
  const file = event.target.files[0];
  if(!file) return;
  const reader = new FileReader();
  reader.onload = function() {
    document.getElementById('profilePic').src = reader.result;
  }
  reader.readAsDataURL(file);
}

function saveProfile() {
  const users = loadUsers();
  const user = users[currentUser];
  const newBio = document.getElementById('bio').value.trim();
  const newPic = document.getElementById('profilePic').src;
  const newPass = document.getElementById('newPassword').value;

  if(newPass && !validPassword(newPass)) {
    alert('Şifre en az 8 karakter, harf ve sayı içermeli!');
    return;
  }

  user.bio = newBio;
  user.profilePic = newPic;
  if(newPass) user.password = newPass;

  saveUsers(users);
  alert('Profil güncellendi!');
  document.getElementById('newPassword').value = '';
  showProfile(currentUser);
}

// ---------- Mavi Tik Verme ----------

function verifyUser(username) {
  if(!currentUser) return alert('Giriş yapmalısınız');
  const users = loadUsers();
  if(!users[currentUser].canVerifyOthers) return alert('Yetkiniz yok');
  if(!users[username]) return alert('Kullanıcı bulunamadı');
  users[username].isVerified = true;
  saveUsers(users);
  alert(username + ' artık doğrulanmış!');
  showProfile(username);
}

// ---------- Takip Sistemi ----------

function toggleFollow(username) {
  if(username === currentUser) return;
  const users = loadUsers();
  const me = users[currentUser];
  const target = users[username];

  const followingIndex = me.following.indexOf(username);
  if(followingIndex >=0) {
    me.following.splice(followingIndex,1);
    const followerIndex = target.followers.indexOf(currentUser);
    if(followerIndex >=0) target.followers.splice(followerIndex,1);
  } else {
    me.following.push(username);
    target.followers.push(currentUser);
  }
  saveUsers(users);
  renderUsersList();
  renderPosts();
}

// ---------- Gönderi Yönetimi ----------

function createPost(content, imageData) {
  if(!content && !imageData) return alert('Metin veya fotoğraf gerekli');
  let posts = loadPosts();
  posts.unshift({
    id: 'post-' + Date.now(),
    user: currentUser,
    content,
    image: imageData || '',
    time: Date.now(),
    likes: [],
    comments: []
  });
  savePosts(posts);
  renderPosts();
}

function renderPosts() {
  const feedDiv = document.getElementById('feed');
  feedDiv.innerHTML = '';

  const posts = loadPosts();
  const users = loadUsers();

  let filteredPosts = [];
  if(currentFeed === 'following') {
    const me = users[currentUser];
    filteredPosts = posts.filter(p => me.following.includes(p.user) || p.user === currentUser);
  } else {
    filteredPosts = posts;
  }

  filteredPosts.forEach(post => {
    const user = users[post.user];
    const liked = post.likes.includes(currentUser);
    const postDiv = document.createElement('div');
    postDiv.className = 'post';

    postDiv.innerHTML = `
      <div class="post-header">
        <img src="${user.profilePic || 'https://via.placeholder.com/50?text=Profil'}" class="profile-pic" alt="profil" />
        <strong>${post.user}${user.isVerified ? ' <span class="verified">✔️</span>' : ''}</strong>
        <small style="margin-left:auto;">${new Date(post.time).toLocaleString()}</small>
      </div>
      <div class="post-content">${post.content ? escapeHtml(post.content).replace(/\n/g,'<br>') : ''}</div>
      ${post.image ? `<img src="${post.image}" alt="post fotoğraf" />` : ''}
      <div>
        <span class="btn-like ${liked ? 'liked' : ''}" onclick="toggleLike('${post.id}')">❤️ ${post.likes.length}</span>
        <span class="btn-comment" onclick="toggleComments('${post.id}')">💬 ${post.comments.length}</span>
      </div>
      <div class="comments" id="comments-${post.id}" style="display:none;"></div>
    `;
    feedDiv.appendChild(postDiv);
  });

  // Gönderi oluştur formu:
  const createDiv = document.createElement('div');
  createDiv.className = 'post';
  createDiv.innerHTML = `
    <h4>Yeni Gönderi</h4>
    <textarea id="newPostContent" placeholder="Gönderi metni..." rows="3"></textarea>
    <input type="file" id="newPostImageInput" accept="image/*" />
    <button onclick="submitNewPost()">Paylaş</button>
  `;
  feedDiv.prepend(createDiv);
}

function escapeHtml(text) {
  if(!text) return '';
  return text.replace(/[&<>"']/g, function(m) {
    return {'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[m];
  });
}

function submitNewPost() {
  const content = document.getElementById('newPostContent').value.trim();
  const fileInput = document.getElementById('newPostImageInput');
  if(fileInput.files.length > 0) {
    const file = fileInput.files[0];
    const reader = new FileReader();
    reader.onload = function() {
      createPost(content, reader.result);
      document.getElementById('newPostContent').value = '';
      fileInput.value = '';
    }
    reader.readAsDataURL(file);
  } else {
    createPost(content, '');
    document.getElementById('newPostContent').value = '';
  }
}

// ---------- Beğeni ----------

function toggleLike(postId) {
  let posts = loadPosts();
  const post = posts.find(p => p.id === postId);
  if(!post) return;
  const likedIndex = post.likes.indexOf(currentUser);
  if(likedIndex >= 0) post.likes.splice(likedIndex, 1);
  else post.likes.push(currentUser);
  savePosts(posts);
  renderPosts();
}

// ---------- Yorumlar ----------

function toggleComments(postId) {
  const commentsDiv = document.getElementById('comments-'+postId);
  if(!commentsDiv) return;
  if(commentsDiv.style.display === 'block') {
    commentsDiv.style.display = 'none';
  } else {
    commentsDiv.style.display = 'block';
    renderComments(postId);
  }
}

function renderComments(postId) {
  const commentsDiv = document.getElementById('comments-'+postId);
  if(!commentsDiv) return;
  commentsDiv.innerHTML = '';

  const posts = loadPosts();
  const post = posts.find(p => p.id === postId);
  if(!post) return;
  const users = loadUsers();

  post.comments.forEach((comment, i) => {
    const user = users[comment.user];
    const commentDiv = document.createElement('div');
    commentDiv.className = 'comment';
    commentDiv.innerHTML = `<strong>${comment.user}${user.isVerified ? ' <span class="verified">✔️</span>' : ''}</strong>: ${escapeHtml(comment.text)}`;
    commentsDiv.appendChild(commentDiv);

    // Yorum yanıtları
    if(comment.replies) {
      comment.replies.forEach(reply => {
        const ru = users[reply.user];
        const replyDiv = document.createElement('div');
        replyDiv.className = 'reply';
        replyDiv.innerHTML = `<strong>${reply.user}${ru && ru.isVerified ? ' <span class="verified">✔️</span>' : ''}</strong>: ${escapeHtml(reply.text)}`;
        commentsDiv.appendChild(replyDiv);
      });
    }

    // Yorum yapma alanı (yalnızca giriş yapan)
    if(currentUser) {
      const replyInput = document.createElement('textarea');
      replyInput.rows = 2;
      replyInput.placeholder = "Yanıt yaz...";
      replyInput.style.width = "95%";
      replyInput.id = `reply-input-${postId}-${i}`;

      const replyBtn = document.createElement('button');
      replyBtn.textContent = "Yanıtla";
      replyBtn.onclick = () => {
        const val = replyInput.value.trim();
        if(val) {
          addReply(postId, i, val);
          renderComments(postId);
        }
      };

      commentsDiv.appendChild(replyInput);
      commentsDiv.appendChild(replyBtn);
    }
  });

  // Yeni yorum ekleme alanı
  if(currentUser) {
    const newCommentInput = document.createElement('textarea');
    newCommentInput.rows = 2;
    newCommentInput.placeholder = "Yorum yaz...";
    newCommentInput.id = `new-comment-${postId}`;
    newCommentInput.style.width = "95%";

    const newCommentBtn = document.createElement('button');
    newCommentBtn.textContent = "Yorum Ekle";
    newCommentBtn.onclick = () => {
      const val = newCommentInput.value.trim();
      if(val) {
        addComment(postId, val);
        renderComments(postId);
      }
    };
    commentsDiv.appendChild(newCommentInput);
    commentsDiv.appendChild(newCommentBtn);
  }
}

function addComment(postId, text) {
  const posts = loadPosts();
  const post = posts.find(p => p.id === postId);
  if(!post) return;
  post.comments.push({user: currentUser, text, replies: []});
  savePosts(posts);
  renderPosts();
}

function addReply(postId, commentIndex, text) {
  const posts = loadPosts();
  const post = posts.find(p => p.id === postId);
  if(!post) return;
  if(!post.comments[commentIndex].replies) post.comments[commentIndex].replies = [];
  post.comments[commentIndex].replies.push({user: currentUser, text});
  savePosts(posts);
  renderPosts();
}

// ---------- Kullanıcı Listesi ----------

function showUsers() {
  document.getElementById('feed').style.display = 'none';
  document.getElementById('profile').style.display = 'none';
  document.getElementById('users').style.display = 'block';
  document.getElementById('messages').style.display = 'none';
  renderUsersList();
}

function renderUsersList() {
  const userListDiv = document.getElementById('userList');
  userListDiv.innerHTML = '';
  const users = loadUsers();

  Object.keys(users).forEach(u => {
    if(u === currentUser) return;
    const user = users[u];
    const div = document.createElement('div');
    div.style.borderBottom = '1px solid #ccc';
    div.style.padding = '8px 0';

    const isFollowing = users[currentUser].following.includes(u);

    div.innerHTML = `
      <img src="${user.profilePic || 'https://via.placeholder.com/40?text=Profil'}" class="profile-pic" style="width:40px; height:40px; vertical-align:middle;" />
      <strong>${u}${user.isVerified ? ' <span class="verified">✔️</span>' : ''}</strong> 
      <button style="float:right; width:100px;" onclick="toggleFollow('${u}')">
        ${isFollowing ? 'Takibi Bırak' : 'Takip Et'}
      </button>
      <button style="float:right; margin-right: 8px; width:100px;" onclick="showProfile('${u}')">Profil</button>
    `;
    userListDiv.appendChild(div);
  });
}

// ---------- Akış Gösterimi ----------

function showFeed(type) {
  currentFeed = type;
  document.getElementById('feed').style.display = 'block';
  document.getElementById('profile').style.display = 'none';
  document.getElementById('users').style.display = 'none';
  document.getElementById('messages').style.display = 'none';
  renderPosts();
}

// ---------- Mesajlaşma ----------

function loadUserSelect() {
  const users = loadUsers();
  const select = document.getElementById('chatUserSelect');
  select.innerHTML = '<option value="">Kullanıcı Seç</option>';
  Object.keys(users).forEach(u => {
    if(u === currentUser) return;
    const option = document.createElement('option');
    option.value = u;
    option.textContent = u + (users[u].isVerified ? ' ✔️' : '');
    select.appendChild(option);
  });
}

function loadChat(toUser) {
  currentChatUser = toUser;
  const messagesDiv = document.getElementById('messages');
  if(!toUser) {
    document.getElementById('messageList').innerHTML = '';
    return;
  }
  document.getElementById('feed').style.display = 'none';
  document.getElementById('profile').style.display = 'none';
  document.getElementById('users').style.display = 'none';
  messagesDiv.style.display = 'block';

  const users = loadUsers();
  const msgs1 = users[currentUser].messages[toUser] || [];
  const msgs2 = users[toUser].messages[currentUser] || [];
  // Mesajları tarih sırasına göre karışık al (iki taraftan)
  let allMsgs = msgs1.concat(msgs2);
  allMsgs.sort((a,b)=>a.time - b.time);

  const listDiv = document.getElementById('messageList');
  listDiv.innerHTML = '';
  allMsgs.forEach(m=>{
    const div = document.createElement('div');
    div.className = 'message ' + (m.from === currentUser ? 'self' : 'other');
    div.textContent = m.text;
    listDiv.appendChild(div);
  });
  listDiv.scrollTop = listDiv.scrollHeight;
}

function sendMessage() {
  const text = document.getElementById('messageInput').value.trim();
  if(!text) return;
  if(!currentChatUser) return alert('Mesaj göndermek için kullanıcı seçin!');
  const users = loadUsers();
  const now = Date.now();
  if(!users[currentUser].messages[currentChatUser]) users[currentUser].messages[currentChatUser] = [];
  users[currentUser].messages[currentChatUser].push({from: currentUser, to: currentChatUser, text, time: now});
  saveUsers(users);
  document.getElementById('messageInput').value = '';
  loadChat(currentChatUser);
}

// ---------- Sayfa Yüklenince ----------

window.onload = () => {
  init();
};
</script>

</body>
</html>

