// Backend: server.js
const express = require('express');
const mongoose = require('mongoose');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const cors = require('cors');

const app = express();
app.use(express.json());
app.use(cors());

// Connect to MongoDB
mongoose.connect('mongodb://localhost:27017/mern-friends', { useNewUrlParser: true, useUnifiedTopology: true });

// Define User schema and model
const userSchema = new mongoose.Schema({
  username: String,
  password: String,
  friends: [String],
  pendingRequests: [String],
});
const User = mongoose.model('User', userSchema);

// Route for user registration
app.post('/signup', async (req, res) => {
  const { username, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10); // Hash the password for security
  const user = new User({ username, password: hashedPassword, friends: [], pendingRequests: [] });
  await user.save();
  res.sendStatus(201); // User created
});

// Route for user login
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  const user = await User.findOne({ username });
  if (user && (await bcrypt.compare(password, user.password))) {
    const token = jwt.sign({ username }, 'secret', { expiresIn: '1h' }); // Generate a JWT token
    res.json({ token });
  } else {
    res.sendStatus(401); // Unauthorized
  }
});

// Fetch all users
app.get('/users', async (req, res) => {
  const users = await User.find({}, 'username friends pendingRequests');
  res.json(users);
});

// Send a friend request
app.post('/send-friend-request', async (req, res) => {
  const { username, targetUsername } = req.body;
  const targetUser = await User.findOne({ username: targetUsername });
  if (targetUser && !targetUser.pendingRequests.includes(username)) {
    targetUser.pendingRequests.push(username); // Add the request to the target user's pendingRequests
    await targetUser.save();
    res.sendStatus(200);
  } else {
    res.sendStatus(400); // Bad Request
  }
});

// Respond to a friend request
app.post('/respond-friend-request', async (req, res) => {
  const { username, targetUsername, accept } = req.body;
  const user = await User.findOne({ username });
  const targetUser = await User.findOne({ username: targetUsername });
  if (user && targetUser) {
    user.pendingRequests = user.pendingRequests.filter((req) => req !== targetUsername);
    if (accept) {
      user.friends.push(targetUsername); // Add to friends list if accepted
      targetUser.friends.push(username);
    }
    await user.save();
    await targetUser.save();
    res.sendStatus(200);
  } else {
    res.sendStatus(400); // Bad Request
  }
});

// Get friend requests for a user
app.get('/friend-requests/:username', async (req, res) => {
  const { username } = req.params;
  const user = await User.findOne({ username });
  if (user) {
    res.json(user.pendingRequests);
  } else {
    res.sendStatus(404); // Not Found
  }
});

// Get friend recommendations based on mutual friends
app.get('/friend-recommendations/:username', async (req, res) => {
  const { username } = req.params;
  const user = await User.findOne({ username });
  if (user) {
    const recommendations = await User.find({ username: { $ne: username } });
    const mutualFriends = recommendations.map((recommendation) => ({
      username: recommendation.username,
      mutual: recommendation.friends.filter((friend) => user.friends.includes(friend)).length,
    }));
    mutualFriends.sort((a, b) => b.mutual - a.mutual); // Sort by number of mutual friends
    res.json(mutualFriends);
  } else {
    res.sendStatus(404); // Not Found
  }
});

// Start the server
app.listen(5000, () => console.log('Server running on port 5000'));

// Frontend: App.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function App() {
  const [users, setUsers] = useState([]);
  const [friends, setFriends] = useState([]);
  const [search, setSearch] = useState('');
  const [recommendations, setRecommendations] = useState([]);
  const [friendRequests, setFriendRequests] = useState([]);

  // Fetch all users and friend requests on component mount
  useEffect(() => {
    axios.get('http://localhost:5000/users').then((res) => {
      setUsers(res.data);
    });
    axios.get('http://localhost:5000/friend-requests/currentUser').then((res) => {
      setFriendRequests(res.data);
    });
  }, []);

  // Fetch friend recommendations
  useEffect(() => {
    axios.get('http://localhost:5000/friend-recommendations/currentUser').then((res) => {
      setRecommendations(res.data);
    });
  }, []);

  // Send a friend request
  const sendFriendRequest = (targetUsername) => {
    axios.post('http://localhost:5000/send-friend-request', { username: 'currentUser', targetUsername });
  };

  // Respond to a friend request
  const respondToFriendRequest = (targetUsername, accept) => {
    axios.post('http://localhost:5000/respond-friend-request', { username: 'currentUser', targetUsername, accept }).then(() => {
      setFriendRequests(friendRequests.filter((req) => req !== targetUsername));
      if (accept) {
        setFriends([...friends, targetUsername]);
      }
    });
  };

  return (
    <div>
      <h1>Friend Finder</h1>
      <input
        type="text"
        placeholder="Search users"
        value={search}
        onChange={(e) => setSearch(e.target.value)}
      />
      <h2>All Users</h2>
      <ul>
        {users
          .filter((u) => u.username.includes(search))
          .map((u) => (
            <li key={u.username}>
              {u.username}{' '}
              {!friends.includes(u.username) && (
                <button onClick={() => sendFriendRequest(u.username)}>Send Friend Request</button>
              )}
            </li>
          ))}
      </ul>
      <h2>Friend Requests</h2>
      <ul>
        {friendRequests.map((req) => (
          <li key={req}>
            {req}{' '}
            <button onClick={() => respondToFriendRequest(req, true)}>Accept</button>
            <button onClick={() => respondToFriendRequest(req, false)}>Reject</button>
          </li>
        ))}
      </ul>
      <h2>Friend Recommendations</h2>
      <ul>
        {recommendations.map((rec) => (
          <li key={rec.username}>
            {rec.username} ({rec.mutual} mutual friends)
            {!friends.includes(rec.username) && (
              <button onClick={() => sendFriendRequest(rec.username)}>Send Friend Request</button>
            )}
          </li>
        ))}
      </ul>
    </div>
  );
}

export default App;
