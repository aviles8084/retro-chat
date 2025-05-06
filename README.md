# retro-chat 


// Chat con salas, usuarios y persistencia en MongoDB
// Requiere: npm install express socket.io cors mongoose bcrypt jsonwebtoken react-router-dom axios

// backend/server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const cors = require('cors');
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

const app = express();
app.use(cors());
app.use(express.json());
const server = http.createServer(app);
const io = new Server(server, {
  cors: { origin: '*' }
});

mongoose.connect('mongodb://localhost:27017/chat_app', {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

const userSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  password: String,
  avatar: String,
  interests: [String]
});
const User = mongoose.model('User', userSchema);

const messageSchema = new mongoose.Schema({
  room: String,
  user: String,
  message: String,
  timestamp: { type: Date, default: Date.now }
});
const Message = mongoose.model('Message', messageSchema);

// Registro
app.post('/register', async (req, res) => {
  const { name, email, password, avatar, interests } = req.body;
  const hashed = await bcrypt.hash(password, 10);
  try {
    const user = new User({ name, email, password: hashed, avatar, interests });
    await user.save();
    res.json({ message: 'Usuario registrado' });
  } catch (e) {
    res.status(400).json({ error: 'Correo ya registrado' });
  }
});

// Login
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user) return res.status(400).json({ error: 'Usuario no encontrado' });
  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) return res.status(400).json({ error: 'Contraseña incorrecta' });

  const token = jwt.sign({ id: user._id, name: user.name }, 'secreto');
  res.json({ token, user: { name: user.name, avatar: user.avatar, interests: user.interests } });
});

io.on('connection', (socket) => {
  console.log('Usuario conectado:', socket.id);

  socket.on('join_room', async (room) => {
    socket.join(room);
    const messages = await Message.find({ room }).sort({ timestamp: 1 });
    socket.emit('chat_history', messages);
  });

  socket.on('send_message', async ({ room, message, user }) => {
    const newMessage = new Message({ room, message, user });
    await newMessage.save();
    io.to(room).emit('receive_message', { message, user });
  });
});

server.listen(3001, () => {
  console.log('Servidor corriendo en puerto 3001');
});

// frontend/src/App.jsx
import { useEffect, useState } from 'react';
import io from 'socket.io-client';
import axios from 'axios';

const socket = io('http://localhost:3001');

function App() {
  const [room, setRoom] = useState('general');
  const [message, setMessage] = useState('');
  const [chat, setChat] = useState([]);
  const [user, setUser] = useState(() => JSON.parse(localStorage.getItem('user')) || null);

  useEffect(() => {
    socket.emit('join_room', room);
    setChat([]);
  }, [room]);

  useEffect(() => {
    socket.on('chat_history', (messages) => {
      setChat(messages.map((m) => `${m.user}: ${m.message}`));
    });

    socket.on('receive_message', (data) => {
      setChat((prev) => [...prev, `${data.user}: ${data.message}`]);
    });

    return () => {
      socket.off('chat_history');
      socket.off('receive_message');
    };
  }, []);

  const sendMessage = () => {
    if (!user) return;
    socket.emit('send_message', { room, message, user: user.name });
    setMessage('');
  };

  const handleLogin = async () => {
    const email = prompt('Email');
    const password = prompt('Contraseña');
    const res = await axios.post('http://localhost:3001/login', { email, password });
    localStorage.setItem('user', JSON.stringify(res.data.user));
    setUser(res.data.user);
  };

  return (
    <div className="p-4">
      <h1 className="text-2xl font-bold mb-4">Sala de Chat: {room}</h1>

      {!user ? (
        <button onClick={handleLogin} className="bg-green-500 text-white px-4 py-2 mb-4">Iniciar Sesión</button>
      ) : (
        <div className="mb-4">Bienvenido, <b>{user.name}</b></div>
      )}

      <div className="mb-4">
        <label className="mr-2 font-bold">Cambiar Sala:</label>
        <select onChange={(e) => setRoom(e.target.value)} value={room} className="border p-1">
          <option value="general">General</option>
          <option value="amistad">Amistad</option>
          <option value="pareja">Pareja</option>
          <option value="hobbies">Hobbies</option>
        </select>
      </div>

      <div className="border p-4 h-64 overflow-y-auto bg-gray-100">
        {chat.map((msg, index) => (
          <div key={index} className="mb-2">{msg}</div>
        ))}
      </div>

      <input
        type="text"
        value={message}
        onChange={(e) => setMessage(e.target.value)}
        className="border p-2 w-full mt-4"
        placeholder="Escribe tu mensaje..."
      />
      <button
        onClick={sendMessage}
        className="bg-blue-500 text-white p-2 mt-2 w-full rounded"
      >
        Enviar
      </button>
    </div>
  );
}

export default App;
