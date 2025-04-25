// server.js
const express = require("express");
const bodyParser = require("body-parser");
const cors = require("cors");
const { Configuration, OpenAIApi } = require("openai");
const admin = require("firebase-admin");
require("dotenv").config();

const serviceAccount = require("./firebase-service-account.json");
admin.initializeApp({ credential: admin.credential.cert(serviceAccount) });
const db = admin.firestore();

const app = express();
const port = process.env.PORT || 3001;

app.use(cors());
app.use(bodyParser.json());

const openai = new OpenAIApi(new Configuration({ apiKey: process.env.OPENAI_API_KEY }));

async function translateContent(content, targetLang = "en") {
  const response = await openai.createChatCompletion({
    model: "gpt-4",
    messages: [
      { role: "system", content: `Translate the following post into ${targetLang}.` },
      { role: "user", content },
    ],
  });
  return response.data.choices[0].message.content;
}

async function moderateContent(content) {
  const response = await openai.createModeration({ input: content });
  return response.data.results[0];
}

app.post("/api/register", async (req, res) => {
  const { commitment } = req.body;
  try {
    await db.collection("users").add({ commitment });
    res.status(200).json({ message: "Identity registered." });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Registration failed." });
  }
});

app.post("/api/post", async (req, res) => {
  try {
    const { content, username, timestamp, lang } = req.body;
    const translated = lang !== "en" ? await translateContent(content, "en") : content;
    const moderation = await moderateContent(translated);
    if (moderation.flagged) {
      return res.status(403).json({ error: "Content violates community guidelines." });
    }
    const postRef = await db.collection("posts").add({ content, username, timestamp, translated });
    res.json({ content, username, timestamp, translated });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Server error." });
  }
});

app.get("/api/feed", async (req, res) => {
  try {
    const snapshot = await db.collection("posts").orderBy("timestamp", "desc").limit(50).get();
    const posts = snapshot.docs.map(doc => doc.data());
    res.json(posts);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Failed to fetch feed." });
  }
});

app.listen(port, () => {
  console.log(`Server running on http://localhost:${port}`);
});
