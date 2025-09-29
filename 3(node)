
const express = require('express');
const app = express();
const port = 3000;

// Middleware to parse JSON bodies
app.use(express.json());

// In-memory data structures
const seats = {};
const locks = {};
const LOCK_DURATION = 60000; // 1 minute in milliseconds

// Initialize seats (5 seats for demo)
for (let i = 1; i <= 5; i++) {
  seats[i] = {
    id: i,
    status: 'available' // available, locked, booked
  };
}

// Helper function to clean expired locks
function cleanExpiredLocks() {
  const now = Date.now();
  for (const seatId in locks) {
    if (locks[seatId].expiresAt <= now) {
      // Lock has expired, make seat available again
      seats[seatId].status = 'available';
      delete locks[seatId];
      console.log(`Lock expired for seat ${seatId}, making it available again`);
    }
  }
}

// Clean expired locks every 30 seconds
setInterval(cleanExpiredLocks, 30000);

// GET /seats - View all seats and their status
app.get('/seats', (req, res) => {
  cleanExpiredLocks(); // Clean before sending response
  
  const seatsWithLockInfo = {};
  for (const seatId in seats) {
    seatsWithLockInfo[seatId] = {
      ...seats[seatId],
      lockedUntil: locks[seatId] ? new Date(locks[seatId].expiresAt).toISOString() : null,
      lockedBy: locks[seatId] ? locks[seatId].userId : null
    };
  }
  
  res.json(seatsWithLockInfo);
});

// POST /lock/:seatId - Lock a seat for a user
app.post('/lock/:seatId', (req, res) => {
  const seatId = req.params.seatId;
  const { userId } = req.body;
  
  if (!userId) {
    return res.status(400).json({
      success: false,
      message: 'User ID is required'
    });
  }
  
  if (!seats[seatId]) {
    return res.status(404).json({
      success: false,
      message: 'Seat not found'
    });
  }
  
  cleanExpiredLocks(); // Clean expired locks first
  
  const seat = seats[seatId];
  
  // Check if seat is already booked
  if (seat.status === 'booked') {
    return res.status(400).json({
      success: false,
      message: 'Seat is already booked'
    });
  }
  
  // Check if seat is currently locked by another user
  if (seat.status === 'locked' && locks[seatId] && locks[seatId].userId !== userId) {
    const lockExpiresIn = Math.max(0, locks[seatId].expiresAt - Date.now());
    return res.status(400).json({
      success: false,
      message: 'Seat is currently locked by another user',
      lockExpiresIn: Math.ceil(lockExpiresIn / 1000) // seconds
    });
  }
  
  // Lock the seat
  const expiresAt = Date.now() + LOCK_DURATION;
  seats[seatId].status = 'locked';
  locks[seatId] = {
    userId: userId,
    lockedAt: Date.now(),
    expiresAt: expiresAt
  };
  
  console.log(`Seat ${seatId} locked by user ${userId} until ${new Date(expiresAt).toISOString()}`);
  
  res.json({
    success: true,
    message: `Seat ${seatId} locked successfully. Confirm within 1 minute.`,
    expiresAt: new Date(expiresAt).toISOString(),
    expiresIn: LOCK_DURATION / 1000 // seconds
  });
});

// POST /confirm/:seatId - Confirm booking for a locked seat
app.post('/confirm/:seatId', (req, res) => {
  const seatId = req.params.seatId;
  const { userId } = req.body;
  
  if (!userId) {
    return res.status(400).json({
      success: false,
      message: 'User ID is required'
    });
  }
  
  if (!seats[seatId]) {
    return res.status(404).json({
      success: false,
      message: 'Seat not found'
    });
  }
  
  cleanExpiredLocks(); // Clean expired locks first
  
  const seat = seats[seatId];
  const lock = locks[seatId];
  
  // Check if seat is locked and by the same user
  if (seat.status !== 'locked' || !lock || lock.userId !== userId) {
    return res.status(400).json({
      success: false,
      message: 'Seat is not locked or not locked by this user'
    });
  }
  
  // Confirm the booking
  seats[seatId].status = 'booked';
  seats[seatId].bookedBy = userId;
  seats[seatId].bookedAt = new Date().toISOString();
  
  // Remove the lock
  delete locks[seatId];
  
  console.log(`Seat ${seatId} booked successfully by user ${userId}`);
  
  res.json({
    success: true,
    message: `Seat ${seatId} booked successfully!`,
    bookedAt: seats[seatId].bookedAt
  });
});

// POST /release/:seatId - Release a lock (optional endpoint for testing)
app.post('/release/:seatId', (req, res) => {
  const seatId = req.params.seatId;
  const { userId } = req.body;
  
  if (!userId) {
    return res.status(400).json({
      success: false,
      message: 'User ID is required'
    });
  }
  
  if (!seats[seatId]) {
    return res.status(404).json({
      success: false,
      message: 'Seat not found'
    });
  }
  
  const seat = seats[seatId];
  const lock = locks[seatId];
  
  // Check if seat is locked by this user
  if (seat.status !== 'locked' || !lock || lock.userId !== userId) {
    return res.status(400).json({
      success: false,
      message: 'Seat is not locked or not locked by this user'
    });
  }
  
  // Release the lock
  seats[seatId].status = 'available';
  delete locks[seatId];
  
  console.log(`Lock released for seat ${seatId} by user ${userId}`);
  
  res.json({
    success: true,
    message: `Lock released for seat ${seatId}`
  });
});

// GET /status - Get system status
app.get('/status', (req, res) => {
  cleanExpiredLocks();
  
  const availableSeats = Object.values(seats).filter(seat => seat.status === 'available').length;
  const lockedSeats = Object.values(seats).filter(seat => seat.status === 'locked').length;
  const bookedSeats = Object.values(seats).filter(seat => seat.status === 'booked').length;
  
  res.json({
    totalSeats: Object.keys(seats).length,
    available: availableSeats,
    locked: lockedSeats,
    booked: bookedSeats,
    activeLocks: Object.keys(locks).length
  });
});

// Error handling middleware
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({
    success: false,
    message: 'Internal server error'
  });
});

// Start server
app.listen(port, () => {
  console.log(`Ticket booking system running on http://localhost:${port}`);
  console.log('\nAvailable endpoints:');
  console.log('GET  /seats         - View all seats');
  console.log('POST /lock/:seatId   - Lock a seat (requires userId in body)');
  console.log('POST /confirm/:seatId - Confirm booking (requires userId in body)');
  console.log('POST /release/:seatId - Release lock (requires userId in body)');
  console.log('GET  /status        - Get system status');
  console.log('\nExample usage:');
  console.log('curl -X POST http://localhost:3000/lock/1 -H "Content-Type: application/json" -d \'{"userId":"user123"}\'');
});

module.exports = app;
