# myexpress
## webhook
https://traipat.csbootstrap.com/webhook

## dependencies


## index 
// index.js
require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const { createClient } = require("@supabase/supabase-js");

const app = express();

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_KEY
);



// ตั้งค่าจาก LINE Developers Console
const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN || "",
  channelSecret: process.env.LINE_CHANNEL_SECRET || ""
};

app.use('/webhook', line.middleware(config));

// รับ webhook
app.post('/webhook', (req, res) => {
  Promise
    .all(req.body.events.map(handleEvent))
    .then(result => res.json(result));
});

// ตอบกลับข้อความ
function handleEvent(event) {
  if (event.type !== 'message' || event.message.type !== 'text') {
    return Promise.resolve(null);
  }

 // return client.replyMessage(event.replyToken, {
  //   type: 'text',
  //   text: `คุณพิมพ์ว่า: ${event.message.text} ใช่ไหม?`
  // });

  // ข้อความที่ผู้ใช้พิมพ์มา
  const userMessage = event.message.text;

  // ข้อความที่ตอบกลับ
  const replyContent = `คุณพิมพ์ว่า: ${userMessage} ใช่ไหม?`;

  return supabase
    .from("messages")
    .insert({
      user_id: event.source.userId,
      message_id: event.message.id,
      type: event.message.type,
      content: userMessage,
      reply_token: event.replyToken,
      reply_content: replyContent,
    })
    .then(({ error }) => {
      if (error) {
        console.error("Error inserting message:", error);
        return client.replyMessage(event.replyToken, {
          type: "text",
          text: "เกิดข้อผิดพลาดในการบันทึกข้อความ",
        });
      }
      return client.replyMessage(event.replyToken, {
        type: "text",
        text: replyContent,
      });
    });

}

const client = new line.Client(config);

app.get('/', (req, res) => {
  res.send('hello world, Traipat');
});

const PORT = process.env.PORT || 3015;
app.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);

});



# code 5/9/68
// index.js (ฉบับอัปเกรด Gemini และแก้ไขการเว้นบรรทัด)
require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const { createClient } = require("@supabase/supabase-js");
const { GoogleGenerativeAI } = require('@google/generative-ai');

const app = express();

// --- การตั้งค่าทั้งหมด ---

// Supabase
const supabase = createClient(
  process.env.SUPABASE_URL,
  // process.env.SUPABASE_KEY
  process.env.SUPABASE_SERVICE_ROLE_KEY
);


// LINE
const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN || "",
  channelSecret: process.env.LINE_CHANNEL_SECRET || ""
};
const client = new line.Client(config);

// Gemini
const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);
const model = genAI.getGenerativeModel({ model: 'gemini-1.5-flash' });

// --- Middleware และ Routes ---
app.use('/webhook', line.middleware(config));


app.post('/webhook', (req, res) => {
  Promise
    .all(req.body.events.map(handleEvent))
    .then(result => res.json(result))
    .catch(err => {
      console.error(err);
      res.status(500).end();
    });
});

//เพิ่งเพิ่ม
async function handleImageMessage(event) {
  const messageId = event.message.id;

  try {
    // ดึงไฟล์จาก LINE
    const stream = await client.getMessageContent(messageId);

    // แปลง stream → buffer
    const chunks = [];
    for await (const chunk of stream) {
      chunks.push(chunk);
    }
    const buffer = Buffer.concat(chunks);

    // อัพโหลดเข้า Supabase Storage
    const fileName = `line_images/${messageId}.jpg`;
    const { data, error } = await supabase.storage
      .from("uploads") // ชื่อ bucket
      .upload(fileName, buffer, {
        contentType: "image/jpeg",
        upsert: true, // ถ้ามีไฟล์ชื่อซ้ำ จะเขียนทับ
      });

    if (error) {
      console.error("❌ Upload error:", error);
      return client.replyMessage(event.replyToken, {
        type: "text",
        text: "อัปโหลดรูปไป Supabase ไม่สำเร็จ",
      });
    }

    console.log("✅ Uploaded to Supabase:", data);

    // ตอบกลับ User
    return client.replyMessage(event.replyToken, {
      type: "text",
      text: "📷 ได้รับรูปแล้ว และอัปโหลดไป Supabase สำเร็จ!",
    });
  } catch (err) {
    console.error("❌ Error:", err);
  }
}



async function handleEvent(event) {
 if (event.type === "message" && event.message.type === "image") {
    return handleImageMessage(event);
  }
  if (event.type !== 'message' || event.message.type !== 'text') {
    return Promise.resolve(null);
  }

  const userMessage = event.message.text;

  try {
    const prompt = `คุณคือ AI ผู้ช่วยที่เป็นมิตรและมีไหวพริบ จงตอบกลับข้อความนี้ตรงๆ: "${userMessage}"`;
    
    const result = await model.generateContent(prompt);
    const response = await result.response;

    // --- จุดที่แก้ไข: เพิ่ม .trim() เพื่อตัดบรรทัดและช่องว่างที่ไม่จำเป็นออก ---
    const geminiReply = response.text().trim(); 

    const { error } = await supabase
      .from("messages")
      .insert({
        user_id: event.source.userId,
        message_id: event.message.id,
        type: event.message.type,
        content: userMessage,
        reply_token: event.replyToken,
        reply_content: geminiReply,
      });

    if (error) {
      console.error("Error inserting message to Supabase:", error);
    }

    return client.replyMessage(event.replyToken, {
      type: "text",
      text: geminiReply,
    });

  } catch (err) {
    console.error("Error communicating with Gemini or LINE:", err);
    return client.replyMessage(event.replyToken, {
      type: 'text',
      text: 'ขออภัย, ตอนนี้ AI กำลังประมวลผลผิดพลาดเล็กน้อย ลองอีกครั้งนะ',
    });
  }

}


app.get('/', (req, res) => {
  res.send('hello world, Traipat');
});

const PORT = process.env.PORT || 3015;
app.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);

});
