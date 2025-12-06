# How to create a simple user feedback form using SurveyJS/React and Node.js + MongoDB Backend

When you want to build modern, dynamic forms in a React application, SurveyJS is one of the most flexible libraries you can pick. Instead of manually building input boxes and validation logic, SurveyJS lets you create everything through a clean JSON schema, and the library renders the entire form UI for you. It also exposes several useful events - such as `onComplete` and `onValueChanged`, that make it easy to control what happens when users interact with your survey.

In this article, we will build a User Feedback Survey, integrate it inside React using SurveyJS, and then send all submitted responses to a Node.js + MongoDB backend.

We will also look at customization options, such as themes, question styling, dynamic behavior, and validation rules.

### What you’ll learn

 - Basic SurveyJS usage inside React.
 - The onComplete event to get final results.
 - Sending the data to Node backend and store in mongoDB.

### What We Are Building

A simple survey with these fields:

 - Name (required)
 - Email (required, validated)
 - Rating (1–5)
 - Comments (optional)

Workflow:

1. User fills the form inside the React UI.
2. SurveyJS fires the onComplete event.
3. Frontend sends data to the backend.
4. Node.js API stores survey results in MongoDB.

Below is step by step guide - 

## 1. Installing Required Packages

### Frontend

``` bash
npm create vite@latest surveyjs-react -- --template react
cd surveyjs-react
npm install
npm install survey-react-ui
```

SurveyJS Form Library for React consists of two npm packages: `survey-core` (platform-independent code) and `survey-react-ui` (rendering code). The `survey-core` package will be installed automatically as a dependency.

### Backend

``` bash
npm install express mongoose cors
```

We’ll use Mongoose for the MongoDB schema and queries.


## 2. Frontend (React + SurveyJS)

We will be following below steps to create and render the form in UI.

#### 2.1 Configure Styles

To add SurveyJS themes to your application, create a React component that will render your form or survey and import the Form Library style sheet.

``` jsx
import 'survey-core/survey-core.css';
```

#### 2.2 Create a Model

Define your survey as a JSON schema (questions, question types, etc.). This schema describes:
 - Questions
 - Question types
 - Pages
 - Validation rules
 - Conditional logic
 - UI preferences

Here, we will be adding 4 items - Name, Email, Ratings and Comments. 

``` jsx
// Simple survey JSON schema
const surveyJson = {
    title: "User Feedback Survey",
    elements: [
    {
        type: "text",
        name: "name",
        title: "Your name",
        isRequired: true,
    },
    {
        type: "text",
        name: "email",
        title: "Email",
        inputType: "email",
        isRequired: true,
    },
    {
        type: "rating",
        name: "rating",
        title: "How would you rate us?",
        isRequired: true,
        rateMin: 1,
        rateMax: 5,
    },
    {
        type: "comment",
        name: "comments",
        title: "Any suggestions?",
    },
    ]
};
```

To instantiate a model, you need to pass the model schema to the Model constructor. The model instance can then be later used to render the survey.

``` jsx
import { Model } from 'survey-core';

const surveyJson = { /* ... */ }

export default function SurveyForm() {
  const survey = new Model(surveyJson);

  return "...";
}
```

#### 2.3 Render the Form

Once the model is ready, you simply mount it using the `Survey` component from `survey-react-ui`. Add it to the template, and pass the model instance you created in the previous step to the component's model attribute

``` jsx
import { Survey } from 'survey-react-ui';

const surveyJson = { /* ... */ }

export default function SurveyForm() {
  const survey = new Model(surveyJson);

  return (
    <div style={{ maxWidth: 720, margin: '20px auto' }}>
      <Survey model={survey} />
    </div>
  );
}

```

#### 2.4 Handle Form Completion

After a respondent submits a form, the results are available within the [onComplete](https://surveyjs.io/Documentation/Library?id=surveymodel#onComplete) event handler. If your application has a user identification system, you can add the user ID to the survey results before sending them to the server

Here is how our complete SurveyForm.jsx component will look like.

``` jsx
import { useEffect } from 'react';
import { Model } from 'survey-core';
import { Survey } from 'survey-react-ui';

// Optionally import the default CSS
import 'survey-core/survey-core.css';

// Simple survey JSON schema
const surveyJson = {
    title: "User Feedback Survey",
    elements: [
    {
        type: "text",
        name: "name",
        title: "Your name",
        isRequired: true,
    },
    {
        type: "text",
        name: "email",
        title: "Email",
        inputType: "email",
        isRequired: true,
    },
    {
        type: "rating",
        name: "rating",
        title: "How would you rate us?",
        isRequired: true,
        rateMin: 1,
        rateMax: 5,
    },
    {
        type: "comment",
        name: "comments",
        title: "Any suggestions?",
    },
    ]
};

export default function SurveyForm() {
  // Create the survey model
  const survey = new Model(surveyJson);
  const SURVEY_ID = 1;

  useEffect(() => {
    // onComplete event: fired when user finishes survey and clicks Complete
    survey.onComplete.add(async (sender) => {
        
      // sender.data contains all answers as key-value pairs
      const payload = {
        surveyId: SURVEY_ID, // optional metadata
        answers: sender.data,
      };

      try {
        const resp = await fetch('http://localhost:5000/api/surveys/submit', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(payload),
        });

        if (!resp.ok) {
          // basic user feedback - you can show modal/toast
          console.error('Server returned an error while saving survey results');
          return;
        }

        // you could show a thank-you message or route away
        console.log('Survey saved successfully');
      } catch (err) {
        console.error('Network or server error', err);
      }
    });

    // Clean-up when component unmounts
    return () => {
      survey.onComplete.clear();
      survey.onValueChanged.clear();
    };
  }, [survey]);

  return (
    <div style={{ maxWidth: 720, margin: '20px auto' }}>
      <Survey model={survey} />
    </div>
  );
}

```

If you replicate the code correctly, you should see the following form:

![SurveyJS form](https://github.com/mirraashid/survey-docs/blob/master/SurveyJs1.png)


On submitting the form, you should see the thankyou message - 


![SurveyJS thankyou message](https://github.com/mirraashid/survey-docs/blob/master/Screenshot%202025-12-04%20at%205.35.00%20PM.png)



## 3. MongoDB Backend (Node.js + Express + Mongoose)

Once your SurveyJS form is collecting responses on the frontend, the next step is to store the submitted data in a reliable database. MongoDB pairs very well with survey-style JSON data because it can store flexible, nested objects without requiring rigid schemas.

In this section, we’ll build a clean backend API that receives survey data from the React app and stores it in MongoDB using Node.js, Express, and Mongoose.

We have already installed the required packages - `express`, `cors` and `mongoose` in Step 1 of this tutorial.

#### 3.1 Create the Project Structure

We will be using the below folder structure for our `node.js` backend

``` pgsql
backend/
 ├── models/
 │    └── SurveyResponse.js
 ├── routes/
 │    └── surveyRoutes.js
 ├── server.js
 ├── package.json
```

 - `models/` holds your database schemas.
 - `routes/` contains Express route definitions.
 - `server.js` is your main server entry point.

#### 3.2 Connect to MongoDB (Using Mongoose)

Inside `server.js`, import the required modules and connect to MongoDB. You should note that we have used `cors()` which allows your React frontend (usually running on another port) to talk to this backend. The complete `server.js` file should look like: 

``` js
import express from "express";
import mongoose from "mongoose";
import cors from "cors";
import surveyRoutes from "./routes/surveyRoutes.js";

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect("mongodb://localhost:27017/surveydb")
  .then(() => console.log("MongoDB connected"))
  .catch((err) => console.error("Connection error:", err));

// Routes
app.use("/api/surveys", surveyRoutes);

// Server
app.listen(5000, () => console.log("Server running on port 5000"));

```

#### 3.3 Define a Mongoose Schema and Model

SurveyJS returns responses as plain JSON objects - their structure depends on your survey. Instead of creating rigid fields, storing the whole response as one object gives flexibility to support multiple survey types without refactoring. Inside `models/SurveyResponse.js` :

``` js
import mongoose from "mongoose";

const SurveyResponseSchema = new mongoose.Schema(
  {
    data: {
      type: Object,
      required: true,
    },
    submittedAt: {
      type: Date,
      default: Date.now,
    }
  },
  { collection: "survey_responses" }
);

export default mongoose.model("SurveyResponse", SurveyResponseSchema);
```


#### 3.4 Create an API Route to Save Survey Data

In `routes/surveyRoutes.js`:

``` js
import express from "express";
import SurveyResponse from "../models/SurveyResponse.js";

const router = express.Router();

// POST /api/surveys/submit
router.post("/submit", async (req, res) => {
  try {
    const response = new SurveyResponse({
      data: req.body
    });

    const saved = await response.save();
    res.status(201).json({ message: "Survey saved successfully", saved });
  } catch (error) {
    console.error("Error saving survey:", error);
    res.status(500).json({ error: "Internal server error" });
  }
});

export default router;
```


## Conclusion

SurveyJS, combined with React, Node.js, and MongoDB, provides a clean, flexible, and scalable way to build modern form workflows. This setup lets you design highly customizable forms on the frontend while storing responses efficiently in the backend without rigid schema limitations. SurveyJS’s JSON-based configuration, built-in themes, and powerful event system make form creation faster and more intuitive compared to building UI components manually. On the backend, Node.js and Mongoose ensure the data is handled reliably and remains easy to query or extend. Overall, this stack offers a smooth, developer-friendly approach for collecting and managing structured user input.

