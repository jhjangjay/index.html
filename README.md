export default async function handler(req, res) {
  if (req.method !== "POST") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  try {
    const { question } = req.body;

    // 🔑 Google Gemini API 호출
    const response = await fetch(
      "https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key=AIzaSyCfHHLRT8XLHtEdRhOwaB0WEOr4JyjhXcc",
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          contents: [
            {
              role: "user",
              parts: [{ text: question }],
            },
          ],
        }),
      }
    );

    const data = await response.json();

    if (data?.candidates?.[0]?.content?.parts?.[0]?.text) {
      res.status(200).json({ answer: data.candidates[0].content.parts[0].text });
    } else {
      res.status(200).json({ answer: "AI 응답을 처리하지 못했습니다." });
    }
  } catch (error) {
    console.error("Error:", error);
    res.status(500).json({ error: "서버 내부 오류 발생" });
  }
}
