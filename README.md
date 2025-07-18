# 22VE1A6774
Affordmed
## app.js
  const express = require('express');
const app = express();

const urlRoutes = require('./routes/urlRoutes');
const logger = require('./middleware/logger');

app.use(express.json());     // To read JSON body
app.use(logger);             // Log each request
app.use('/api/url', urlRoutes); // Use routes

const PORT = 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

## middleware/logger.js
  module.exports = (req, res, next) => {
  const log = `[${new Date().toISOString()}] ${req.method} ${req.originalUrl}`;
  require('fs').appendFileSync('logs.txt', log + '\n');
  next(); // move to next middleware or route
};

## utils/generateShortCode.js
 const { nanoid } = require('nanoid');

function generateShortCode() {
  return nanoid(6);
}

module.exports = generateShortCode;

## data/urlStore.js
const urlDatabase = new Map();
module.exports = urlDatabase;

## routes/urlRoutes.js
 const express = require('express');
const router = express.Router();
const { createShortUrl, getStats } = require('../controllers/urlController');
const urlDatabase = require('../data/urlStore');

// Create short URL
router.post('/shorten', createShortUrl);

// Get stats
router.get('/stats/:code', getStats);

// Redirect if code is used
router.get('/:code', (req, res) => {
  const code = req.params.code;
  const data = urlDatabase.get(code);
  if (!data) return res.status(404).send('URL not found');
  if (Date.now() > data.expiresAt) return res.status(410).send('URL expired');
  data.hits++;
  res.redirect(data.longUrl);
});

module.exports = router;

## controllers/urlController.js
 const generateShortCode = require('../utils/generateShortCode');
const urlDatabase = require('../data/urlStore');

const createShortUrl = (req, res) => {
  const { longUrl, validityInMinutes } = req.body;
  if (!longUrl) return res.status(400).json({ error: 'longUrl is required' });

  const shortCode = generateShortCode();
  const now = Date.now();
  const ttl = validityInMinutes ? validityInMinutes * 60000 : 30 * 60000;

  urlDatabase.set(shortCode, {
    longUrl,
    createdAt: now,
    expiresAt: now + ttl,
    hits: 0,
  });

  const shortUrl = `${req.protocol}://${req.get('host')}/${shortCode}`;
  res.json({ shortUrl });
};

const getStats = (req, res) => {
  const code = req.params.code;
  const data = urlDatabase.get(code);
  if (!data) return res.status(404).json({ error: 'Link not found' });
  if (Date.now() > data.expiresAt) return res.status(410).json({ error: 'Link expired' });

  data.hits++;
  res.json({
    originalUrl: data.longUrl,
    hits: data.hits,
    createdAt: new Date(data.createdAt),
    expiresAt: new Date(data.expiresAt),
  });
};

module.exports = { createShortUrl, getStats };
