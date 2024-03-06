# TwelveLabs JavaScript SDK

This SDK provides a convenient way to interact with the Twelve Labs Video Understanding Platform from an application written in JavaScript or TypeScript language. The SDK equips you with a set of intuitive methods that streamline the process of interacting with the platform, minimizing the need for boilerplate code.


# Prerequisites

Ensure that the following prerequisites are met before using the SDK:

-  [Node.js](https://nodejs.org/) 20 or newer must be installed on your machine.
-  You have an API key. If you don't have an account, please [sign up](https://api.twelvelabs.io/) for a free account. Then, to retrieve your API key, go to the [Dashboard](https://api.twelvelabs.io/dashboard) page, and select the **Copy** icon to the right of the key to copy it to your clipboard.

# Install the SDK

Install the `twelvelabs-js` package:

 ```sh
yarn add twelvelabs-js # or npm i twelvelabs-js
 ```

# Initialize the SDK

1. Import the required packages into your application:

   ```js
  import { TwelveLabs, SearchData, Task } from 'twelvelabs-js';
  import { promises as fsPromises } from 'fs';
  import * as path from 'path';
   ```

1.  Instantiate the SDK client with your API key. This example code assumes that your API key is stored in an environment variable named `TL_API_KEY`:

    ```js
    const client = new TwelveLabs({ apiKey: '<YOUR_API_KEY>' });
    ```

# Use the SDK

To get started with the SDK, follow these basic steps:

1. Create an index.
2. Upload videos.
3. Perform downstream tasks, such as searching or generating text from video.

> Note: The SDK uses asynchronous methods to interact with the platform. Place your code within an immediately invoked function expression (IIFE), as shown below:

```js
(async () => {
  // Your code
})();
```

## Create an index

To create an index, use the example code below, replacing '<YOUR_INDEX_NAME>' with the desired name for your index:

```js
let index = await client.index.create({
  name: '<YOUR_INDEX_NAME>',
  engines: [
    {
      name: 'marengo2.6',
      options: ['visual', 'conversation', 'text_in_video'],
    },
    {
      name: 'pegasus1',
      options: ['visual', 'conversation'],
    },
  ],
});

console.log(`Created index: id=${index.id} name=${index.name} engines=${JSON.stringify(index.engines)}`);
```

Note the following about this example:
- The platform provides two distinct engine types - embedding and generative, each serving unique purposes in multimodal video understanding.
  - **Embedding engines (Marengo)**: These engines are proficient at performing tasks such as search and classification, enabling enhanced video understanding.
  - **Generative engines (Pegasus)**: These engines generate text based on your videos.
  For your index, both Marengo and Pegasus are enabled.
- The `engines.options` fields specify the types of information each video understanding engine will process. For details, see the [Engine options](https://docs.twelvelabs.io/v1.2/docs/engine-options) page.
- The engines and the engine options specified when you create an index apply to all the videos you upload to that index and cannot be changed. For details, see the [Engine options](https://docs.twelvelabs.io/v1.2/docs/engine-options) page.

The output should look similar to the following:

```
Created index: id=65e71802bb29f13bdd6f38d8 name=2024-03-05T13:02:57.938Z engines=[{"name":"pegasus1","options":["visual","conversation"]},{"name":"marengo2.6","options":["visual","conversation","text_in_video"]}]
```

Note that the API returns, among other information, a field named `id`, representing the unique identifier of your new index.

For a description of each field in the request and response, see the [Create an index](https://docs.twelvelabs.io/v1.2/reference/create-index) page.


## Upload videos

Before you upload a video to the platform, ensure that it meets the following requirements:

- **Video resolution**: Must be greater or equal than 360p and less or equal than 4K. For consistent search results, Twelve Labs recommends you upload 360p videos.
- **Duration**: For Marengo, it must be between 4 seconds and 2 hours (7,200s). For Pegasus, it must be between 4 seconds and 20 minutes (1200s).
- **File size**: Must not exceed 2 GB. If you require different options, send us an email at support@twelvelabs.io.
- **Audio track**: If the `conversation` [engine option](https://docs.twelvelabs.io/v1.2/docs/engine-options) is selected, the video you're uploading must contain an audio track.

To upload videos, use the example code below, replacing the following:

- **`<YOUR_VIDEO_PATH>`**: with a string representing the path to the directory containing the video files you wish to upload.
- **`<YOUR_INDEX_ID>`**: with a string representing the unique identifier of the index to which you want to upload your video.


```js
const files = await fsPromises.readdir('YOUR_VIDEO_PATH');
for (const file of files) {
  const filePath = path.join(__dirname, file);
  console.log(`Uploading ${filePath}`);
  const task = await client.task.create({
    indexId: '<YOUR_INDEX_ID>'
    file: filePath,
  });
  console.log(`Created task: id=${task.id}`);
  await task.waitForDone(50, (task: Task) => {
    console.log(`  Status=${task.status}`);
  });
  if (task.status !== 'ready') {
    throw new Error(`Indexing failed with status ${task.status}`);
  }
  console.log(`Uploaded ${videoPath}. The unique identifer of your video is ${task.videoId}`);
}
```

Note that once a video has been successfully uploaded and indexed, the response will contain a field named `videoId`, representing the unique identifier of your video.

For a description of each field in the request and response, see the [Create a video indexing task](https://docs.twelvelabs.io/reference/create-video-indexing-task) page.

## Perform downstream tasks

The sections below show how you can perform the most common downstream tasks. See [our documentation](https://docs.twelvelabs.io/docs) for a complete list of all the features the Twelve Labs Understanding Platform provides.

### Search

To perform a search request, use the example code below, replacing the following:

- **`<YOUR_INDEX_ID>`**: with a string representing the unique identifier of your index.
- **`<YOUR_QUERY>`**: with a string representing your search query. Note that the API supports full natural language-based search. The following examples are valid queries: "birds flying near a castle," "sun shining on water," and "an officer holding a child's hand."
- **`[<YOUR_SEARCH_OPTIONS>]`**: with an array of strings that specifies the sources of information the platform uses when performing a search. For example, to search based on visual and conversation cues, use `["visual", "conversation"]`. Note that the search options you specify must be a subset of the engine options used when you created the index. For more details, see the [Search options](https://docs.twelvelabs.io/docs/search-options) page.

```js
let searchResults = await client.search.query({
  indexId: '<YOUR_INDEX_ID>'
  query: '<YOUR_QUERY>',
  options: ['<YOUR_SEARCH_OPTIONS>'],
});
printPage(searchResults.data);
while (true) {
  const page = await searchResults.next();
  if (page === null) break;
  else printPage(page);
}

// Utility function to print a specific page
function printPage(searchData) {
  (searchData as SearchData[]).forEach((clip) => {
    console.log(
      `video_id= ${clip.videoId} score=${clip.score} start=${clip.start} end=${clip.end} confidence=${clip.confidence}`,
    );
  });
}
```

The results are returned one page at a time, with a default limit of 10 results on each page. The `next ` method returns the next page of results. When you've reached the end of the dataset, the response is `null`.

```
 video_id=65ca2bce48db9fa780cb3fa4 score=84.9 start=104.9375 end=111.90625 confidence=high
 video_id=65ca2bce48db9fa780cb3fa4 score=84.82 start=160.46875 end=172.75 confidence=high
 video_id=65ca2bce48db9fa780cb3fa4 score=84.77 start=55.375 end=72.46875 confidence=high
```


Note that the response contains, among other information, the following fields:
- `videoId`: The unique identifier of the video that matched your search terms.
- `score`: A quantitative value determined by the AI engine representing the level of confidence that the results match your search terms.
- `start`: The start time of the matching video clip, expressed in seconds.
- `end`: The end time of the matching video clip, expressed in seconds.
- `confidence`: A qualitative indicator based on the value of the score field. This field can take one of the following values:
  - `high`
  - `medium`
  - `low`
  - `extremely low`


For a description of each field in the request and response, see the [Make a search request](https://docs.twelvelabs.io/v1.2/reference/make-search-request) page.

### Generate text from video

The Twelve Labs Video Understanding Platform offers three distinct endpoints tailored to meet various requirements. Each endpoint has been designed with specific levels of flexibility and customization to accommodate different needs.

Note the following about using these endpoints:
- The Pegasus video understanding engine must be enabled for the index to which your video has been uploaded.
- Your prompts must be instructive or descriptive, and you should not phrase them as questions.
- The maximum length of a prompt is 300 characters.

#### Topics, titles, and hashtags

To generate topics, titles, and hashtags, use the example code below, replacing the following:

- **`<YOUR_VIDEO_ID>`**: with a string representing the unique identifier of your video.
- **`[<TYPES>]`**: with an array of strings representing the type of text the platform should generate. Example: `["title", "topic", "hashtag"]`.

```py
const gist = await client.generate.gist('<YOUR_VIDEO_ID>', ['<TYPES>']);
console.log(`Title: ${gist.title}\nTopics=${gist.topics}\nHashtags=${gist.hashtags}`);
```

For a description of each field in the request and response, see the [Titles, topics, or hashtags](https://docs.twelvelabs.io/v1.2/reference/generate-gist) page.

#### Summaries, chapters, and highlights

To generate summaries, chapters, and highlights, use the example code below, replacing the following:

- **`<YOUR_VIDEO_ID>`**: with a string representing the unique identifier of your video.
- **`<TYPE>`**: with a string representing the type of text the platform should generate. This parameter can take one of the following values: "summary", "chapter", or "highlight".
- _(Optional)_ **`<YOUR_PROMPT>`**: with a string that provides context for the summarization task, such as the target audience, style, tone of voice, and purpose. Example:  "Generate a summary in no more than 5 bullet points."


```js
const summary = await client.generate.summarize('<YOUR_VIDEO_ID>', '<TYPE>');
console.log(`Summary: ${summary.summary}`);
```

For a description of each field in the request and response, see the [Summaries, chapters, or highlights](https://docs.twelvelabs.io/v1.2/docs/generate-summaries-chapters-highlights) page.

#### Open-ended texts

To generate open-ended texts, use the example code below, replacing the following:
- **`<YOUR_VIDEO_ID>`**: with a string representing the unique identifier of your video.
- **`<YOUR_PROMPT>`**: with a string that guides the model on the desired format or content. The maximum length of the prompt is 500 tokens or roughly 350 words. Example:  "I want to generate a description for my video with the following format: Title of the video, followed by a summary in 2-3 sentences, highlighting the main topic, key events, and concluding remarks."

```py
const text = await client.generate.text('<YOUR_VIDEO_ID>', '<YOUR_PROMPT>');
console.log(`Open-ended Text: ${text.data}`);
```

## Error Handling

The SDK includes a set of exceptions that are mapped to specific HTTP status codes, as shown in the table below:

| Exception | HTTP Status Code |
|----------|----------|
| BadRequestError| 400 |
| AuthenticationError | 401 |
| PermissionDeniedError  | 403 |
| NotFoundError | 404 |
| ConflictError | 409 |
| UnprocessableEntityError | 422 |
| RateLimitError | 429 |
| InternalServerError | 5xx |

The following example shows how you can handle specific HTTP errors in your application:

```js
try {
  let index = await client.index.create({
    name: '<YOUR_INDEX_NAME>',
    engines: [
      {
        name: 'marengo2.6',
        options: ['visual', 'conversation'],
      },
    ],
  });
  console.log(`Created index: id=${index.id} name=${index.name} engines=${JSON.stringify(index.engines)}`);
} catch (e) {
  console.log(e);
}
```
