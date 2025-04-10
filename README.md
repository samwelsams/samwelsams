<!DOCTYPE html><html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Super Flip</title>
  <style>
    body { font-family: Arial; background: #f2f2f2; margin: 20px; }
    h1 { color: #333; }
    .video-card { background: white; padding: 15px; margin: 10px 0; border-radius: 8px; box-shadow: 0 0 5px rgba(0,0,0,0.1); }
    video { max-width: 100%; height: auto; border-radius: 8px; }
    button { margin-top: 10px; }
    input, button, textarea { padding: 6px; margin: 5px 0; }
    .comments { margin-top: 10px; font-size: 0.9em; }
    .comment { padding: 5px; border-bottom: 1px solid #ccc; }
  </style>
</head>
<body>
  <h1>Super Flip</h1>  <!-- Auth Section -->  <h2>Register / Login</h2>
  <input type="email" id="email" placeholder="Email" />
  <input type="password" id="password" placeholder="Password" />
  <button onclick="register()">Register</button>
  <button onclick="login()">Login</button>
  <p id="auth-status"></p>  <!-- Upload Section -->  <h2>Upload a Video</h2>
  <form id="uploadForm">
    <input type="text" id="title" placeholder="Video title" required />
    <input type="file" id="videoFile" accept="video/*" required />
    <button type="submit">Upload</button>
  </form>  <!-- Feed Section -->  <h2>Feed</h2>
  <div id="feed"></div>  <!-- Screen Sharing -->  <h2>Live Screen Sharing</h2>
  <button onclick="startSharing()">Start Sharing Screen</button>
  <button onclick="viewScreen()">View Shared Screen</button>
  <video id="screenViewer" autoplay muted></video>  <script>
    const API_BASE = 'http://localhost:8000';
    let accessToken = '';

    async function register() {
      const email = document.getElementById('email').value;
      const password = document.getElementById('password').value;
      const formData = new FormData();
      formData.append('email', email);
      formData.append('password', password);
      const res = await fetch(`${API_BASE}/register`, { method: 'POST', body: formData });
      document.getElementById('auth-status').innerText = res.ok ? 'Registered!' : 'Register failed';
    }

    async function login() {
      const formData = new FormData();
      formData.append('username', document.getElementById('email').value);
      formData.append('password', document.getElementById('password').value);
      const res = await fetch(`${API_BASE}/token`, { method: 'POST', body: formData });
      if (res.ok) {
        const data = await res.json();
        accessToken = data.access_token;
        document.getElementById('auth-status').innerText = 'Logged in!';
      } else {
        document.getElementById('auth-status').innerText = 'Login failed';
      }
    }

    async function fetchFeed() {
      const res = await fetch(`${API_BASE}/feed`);
      const videos = await res.json();
      const feed = document.getElementById('feed');
      feed.innerHTML = '';
      for (let v of videos) {
        const div = document.createElement('div');
        div.className = 'video-card';
        div.innerHTML = `
          <h3>${v.title}</h3>
          <video src="${API_BASE}/videos/${v.filename}" controls></video><br>
          <button onclick="likeVideo('${v.id}')">Like (${v.likes})</button>
          <div class="comments" id="comments-${v.id}"></div>
          <textarea id="comment-input-${v.id}" rows="2" placeholder="Write a comment..."></textarea><br><button onclick="postComment('${v.id}')">Post Comment</button>
    `;
    feed.appendChild(div);
    loadComments(v.id);
  }
}

async function likeVideo(id) {
  if (!accessToken) return alert('Login first to like');
  await fetch(`${API_BASE}/like/${id}`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${accessToken}` }
  });
  fetchFeed();
}

document.getElementById('uploadForm').addEventListener('submit', async (e) => {
  e.preventDefault();
  if (!accessToken) return alert('Login first to upload');
  const formData = new FormData();
  formData.append('title', document.getElementById('title').value);
  formData.append('file', document.getElementById('videoFile').files[0]);
  await fetch(`${API_BASE}/upload`, {
    method: 'POST',
    body: formData,
    headers: { Authorization: `Bearer ${accessToken}` }
  });
  fetchFeed();
});

async function loadComments(videoId) {
  const res = await fetch(`${API_BASE}/comments/${videoId}`);
  const comments = await res.json();
  const container = document.getElementById(`comments-${videoId}`);
  container.innerHTML = '<b>Comments:</b>' + comments.map(c => `<div class="comment">${c.text} <i>(${new Date(c.created_at).toLocaleString()})</i></div>`).join('');
}

async function postComment(videoId) {
  if (!accessToken) return alert('Login to comment');
  const input = document.getElementById(`comment-input-${videoId}`);
  const formData = new FormData();
  formData.append('text', input.value);
  await fetch(`${API_BASE}/comments/${videoId}`, {
    method: 'POST',
    body: formData,
    headers: { Authorization: `Bearer ${accessToken}` }
  });
  input.value = '';
  loadComments(videoId);
}

async function startSharing() {
  const stream = await navigator.mediaDevices.getDisplayMedia({ video: true });
  const socket = new WebSocket(`ws://localhost:8000/ws/share/user123`);
  const mediaRecorder = new MediaRecorder(stream, { mimeType: 'video/webm; codecs=vp8' });
  mediaRecorder.ondataavailable = (e) => {
    if (e.data.size > 0) socket.send(e.data);
  };
  mediaRecorder.start(100);
}

function viewScreen() {
  const video = document.getElementById('screenViewer');
  const socket = new WebSocket(`ws://localhost:8000/ws/view/user123`);
  let mediaSource = new MediaSource();
  video.src = URL.createObjectURL(mediaSource);

  mediaSource.addEventListener('sourceopen', () => {
    const sourceBuffer = mediaSource.addSourceBuffer('video/webm; codecs=vp8');
    socket.onmessage = (event) => {
      if (sourceBuffer.updating || mediaSource.readyState !== 'open') return;
      sourceBuffer.appendBuffer(new Uint8Array(event.data));
    };
  });
}

fetchFeed();

  </script>
</body>
</html>
