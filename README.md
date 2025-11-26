# NLP
NLP with AWS and Nodejs

AWS offers two primary paths for NLP in Node.js: **Generative AI** (using Amazon Bedrock) and **Specialized NLP Services** (using Amazon Comprehend/Transcribe).

For a Senior SDE (SDE 3) context, the most valuable pattern is a serverless event-driven pipeline. Below are detailed examples ranging from direct SDK usage to a complete architectural workflow.

### 1. Generative AI: Text & Chat with Amazon Bedrock
Amazon Bedrock is the modern standard for NLP on AWS, allowing you to invoke foundation models (like Claude 3 or Amazon Nova) directly.

**Key Pattern:** Use the `ConverseCommand` API. It abstracts model-specific payload differences, making your code portable across different models (e.g., swapping Claude for Titan).

```javascript
import { BedrockRuntimeClient, ConverseCommand } from "@aws-sdk/client-bedrock-runtime";

// Initialize Client (SDE 3 Tip: Re-use this client outside the handler in Lambda for connection reuse)
const client = new BedrockRuntimeClient({ region: "us-east-1" });

export const generateResponse = async (userInput) => {
  const modelId = "anthropic.claude-3-haiku-20240307-v1:0"; // High speed, low cost model

  const command = new ConverseCommand({
    modelId,
    messages: [
      {
        role: "user",
        content: [{ text: userInput }],
      },
    ],
    inferenceConfig: {
      maxTokens: 512,
      temperature: 0.5,
    },
  });

  try {
    const response = await client.send(command);
    // Parse the response directly from the standardized output
    const responseText = response.output.message.content[0].text;
    return responseText;
  } catch (err) {
    console.error("Bedrock API Error:", err);
    throw err;
  }
};
```


### 2. Specialized NLP: Sentiment Analysis with Amazon Comprehend
For high-volume, narrow tasks (like analyzing millions of customer reviews), specialized services are often faster and cheaper than a large LLM.

**Use Case:** Detecting negative sentiment in user feedback streams.

```javascript
import { ComprehendClient, DetectSentimentCommand } from "@aws-sdk/client-comprehend";

const client = new ComprehendClient({ region: "us-east-1" });

export const analyzeSentiment = async (text) => {
  const command = new DetectSentimentCommand({
    Text: text,
    LanguageCode: "en", // 'es', 'fr', etc.
  });

  try {
    const response = await client.send(command);
    // Returns: { Sentiment: 'POSITIVE'|'NEGATIVE'|..., SentimentScore: { ... } }
    if (response.Sentiment === 'NEGATIVE' && response.SentimentScore.Negative > 0.9) {
      console.log("Alert: High confidence negative feedback detected.");
    }
    return response;
  } catch (err) {
    console.error("Comprehend Error:", err);
  }
};
```


### 3. SDE 3 Architecture: Serverless Document Processing Pipeline
A common system design interview question involves building an automated pipeline to process documents.

**Scenario:** A user uploads a PDF invoice. You need to extract text, summarize it, and store the metadata.

**Architecture:** `S3 (Upload)` → `Lambda (Trigger)` → `Textract (OCR)` → `Bedrock (Summarize)` → `DynamoDB (Store)`

#### Phase 1: Text Extraction (Lambda A)
Instead of processing everything in one function, use Step Functions or decoupled Lambdas. Here is the extraction logic:

```javascript
import { TextractClient, DetectDocumentTextCommand } from "@aws-sdk/client-textract";

const textract = new TextractClient({});

export const extractText = async (bucket, key) => {
  const command = new DetectDocumentTextCommand({
    Document: {
      S3Object: {
        Bucket: bucket,
        Name: key,
      },
    },
  });

  const response = await textract.send(command);
  // Reduce all line blocks into a single string
  return response.Blocks
    .filter(block => block.BlockType === 'LINE')
    .map(block => block.Text)
    .join('\n');
};
```

#### Phase 2: Intelligent Summarization (Lambda B)
Once you have the raw text, pass it to Bedrock for NLP processing.

```javascript
import { BedrockRuntimeClient, ConverseCommand } from "@aws-sdk/client-bedrock-runtime";

const bedrock = new BedrockRuntimeClient({});

export const summarizeInvoice = async (rawText) => {
  const prompt = `Extract the vendor name, total amount, and invoice date from the following text. Return strictly JSON.\n\n${rawText}`;

  const command = new ConverseCommand({
    modelId: "anthropic.claude-3-haiku-20240307-v1:0",
    messages: [{ role: "user", content: [{ text: prompt }] }],
  });

  const response = await bedrock.send(command);
  return JSON.parse(response.output.message.content[0].text);
};
```

### 4. Real-Time Audio NLP
For real-time speech-to-text (e.g., a voice assistant), you cannot use standard REST APIs. You must use **WebSockets** with `TranscribeStreamingClient`.

**Implementation Note:** This requires signing specific WebSocket frames, which the SDK handles for you.

```javascript
import { TranscribeStreamingClient, StartStreamTranscriptionCommand } from "@aws-sdk/client-transcribe-streaming";

const client = new TranscribeStreamingClient({ region: "us-east-1" });

// In a real app, 'audioStream' would be an async generator yielding audio chunks
const startTranscription = async (audioStream) => {
  const command = new StartStreamTranscriptionCommand({
    LanguageCode: "en-US",
    MediaEncoding: "pcm",
    MediaSampleRateHertz: 44100,
    AudioStream: audioStream, // Async iterable
  });

  const response = await client.send(command);

  // Consume the async iterable response stream
  for await (const event of response.TranscriptResultStream) {
    if (event.TranscriptEvent) {
      const results = event.TranscriptEvent.Transcript.Results;
      if (results.length > 0 && !results[0].IsPartial) {
        console.log("Final Transcript:", results[0].Alternatives[0].Transcript);
      }
    }
  }
};
```


### Summary of Services

| Service | Node.js SDK Client | Best For |
| :--- | :--- | :--- |
| **Amazon Bedrock** | `BedrockRuntimeClient` | Generative tasks, chat, summarization, complex extraction. |
| **Amazon Comprehend** | `ComprehendClient` | Sentiment analysis, PII masking, topic modeling (high speed/volume). |
| **Amazon Transcribe** | `TranscribeStreamingClient` | Real-time or batch speech-to-text conversion. |
| **Amazon Textract** | `TextractClient` | OCR extraction from PDFs and images (documents). |

[1](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/javascript_comprehend_code_examples.html)
[2](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/comprehend)
[3](https://docs.aws.amazon.com/comprehend/latest/dg/how-sentiment.html)
[4](https://getstream.io/blog/aws-comprehend-sentiment-analysis/)
[5](https://www.youtube.com/watch?v=kUyaJZgPmn8)
[6](https://www.microtica.com/blog/amazon-bedrock-a-practical-guide-for-developers-and-devops-engineers)
[7](https://docs.aws.amazon.com/code-library/latest/ug/javascript_3_transcribe_code_examples.html)
[8](https://docs.aws.amazon.com/code-library/latest/ug/javascript_3_comprehend_code_examples.html)
[9](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/javascript_bedrock_code_examples.html)
[10](https://aws.amazon.com/blogs/machine-learning/transcribe-speech-to-text-in-real-time-using-amazon-transcribe-with-websocket/)
[11](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/javascript_code_examples.html)
[12](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/javascript_bedrock-runtime_code_examples.html)
[13](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/javascript_transcribe_code_examples.html)
[14](https://docs.aws.amazon.com/comprehend/latest/dg/example_comprehend_DetectSentiment_section.html)
[15](https://github.com/YAV-AI/amazon-bedrock-node-js-samples)
[16](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/transcribe-examples-section.html)
[17](https://stackoverflow.com/questions/67229000/aws-comprehend-with-aws-lambda-in-node-js)
[18](https://www.raymondcamden.com/2024/04/04/a-quick-first-look-at-amazon-bedrock-with-nodejs)
[19](https://github.com/aws/aws-sdk-js-v3/discussions/6655)
[20](https://www.techtarget.com/searchaws/tip/Amazon-Comprehend-covers-the-basics-of-text-analysis)
