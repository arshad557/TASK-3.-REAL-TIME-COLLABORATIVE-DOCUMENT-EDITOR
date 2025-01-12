# TASK-3.-REAL-TIME-COLLABORATIVE-DOCUMENT-EDITOR
CODETECH Internship 

// Collaborative Document Editor - Real-Time using React.js and Node.js

// BACKEND: server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const mongoose = require('mongoose');
const Document = require('./models/Document');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

// MongoDB connection
mongoose.connect('mongodb://localhost:27017/collab_docs', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

const PORT = process.env.PORT || 4000;

// Serve static files (React frontend)
app.use(express.static('public'));

// WebSocket handling
io.on('connection', (socket) => {
  console.log('A user connected:', socket.id);

  socket.on('get-document', async (documentId) => {
    const document = await findOrCreateDocument(documentId);
    socket.join(documentId);
    socket.emit('load-document', document.data);

    socket.on('send-changes', (delta) => {
      socket.to(documentId).emit('receive-changes', delta);
    });

    socket.on('save-document', async (data) => {
      await Document.findByIdAndUpdate(documentId, { data });
    });
  });

  socket.on('disconnect', () => {
    console.log('User disconnected:', socket.id);
  });
});

async function findOrCreateDocument(id) {
  if (!id) return;
  const existingDoc = await Document.findById(id);
  if (existingDoc) return existingDoc;
  return await Document.create({ _id: id, data: '' });
}

server.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);
});

// MongoDB Schema: models/Document.js
const { Schema, model } = mongoose;
const DocumentSchema = new Schema({
  _id: String,
  data: String,
});
module.exports = model('Document', DocumentSchema);

// FRONTEND: src/App.js (React)
import React, { useEffect, useCallback } from 'react';
import Quill from 'quill';
import { io } from 'socket.io-client';
import 'quill/dist/quill.snow.css';

const SAVE_INTERVAL_MS = 2000;

function App() {
  const socket = io.connect('http://localhost:4000');
  const editorRef = React.useRef();
  const documentId = 'example-document-id'; // Replace with dynamic ID as needed

  useEffect(() => {
    const editor = new Quill('#editor', { theme: 'snow' });
    editorRef.current = editor;

    socket.emit('get-document', documentId);

    socket.on('load-document', (data) => {
      editor.setContents(JSON.parse(data));
    });

    socket.on('receive-changes', (delta) => {
      editor.updateContents(delta);
    });

    const saveInterval = setInterval(() => {
      socket.emit('save-document', JSON.stringify(editor.getContents()));
    }, SAVE_INTERVAL_MS);

    editor.on('text-change', (delta) => {
      socket.emit('send-changes', delta);
    });

    return () => {
      clearInterval(saveInterval);
      socket.disconnect();
    };
  }, [socket, documentId]);

  return (
    <div style={{ padding: '20px' }}>
      <h1>Collaborative Document Editor</h1>
      <div id="editor" style={{ height: '500px', backgroundColor: '#fff' }}></div>
    </div>
  );
}

export default App;

// Run React with: npm start
// Backend: node server.js
