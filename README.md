# Disappearing-Tweet
npx create-react-app real-time-tweet-streamer
cd real-time-tweet-streamer
"scripts": {
  "start": "npm run development",
  "development": "NODE_ENV=development concurrently --kill-others \"npm run client\" \"npm run server\"",
  "production": "npm run build && NODE_ENV=production npm run server",
  "client": "react-scripts start",
  "server": "node server/server.js",
  "build": "react-scripts build",
  "test": "react-scripts test",
  "eject": "react-scripts eject"
},
{
  "name": "real-time-tweet-streamer",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "@testing-library/jest-dom": "^4.2.4",
    "@testing-library/react": "^9.3.2",
    "@testing-library/user-event": "^7.1.2",
    "react": "^16.13.1",
    "react-dom": "^16.13.1",
    "react-scripts": "3.4.1"
  },
  "scripts": {
    "start": "npm run development",
    "development": "NODE_ENV=development concurrently --kill-others \"npm run client\" \"npm run server\"",
    "production": "npm run build && NODE_ENV=production npm run server",
    "client": "react-scripts start",
    "server": "node server/server.js",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": "react-app"
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  }
}
rm src/*
import React from "react";
import ReactDOM from "react-dom";
import App from "./components/App";

ReactDOM.render(<App />, document.querySelector("#root"));
export TWITTER_BEARER_TOKEN=<YOUR BEARER TOKEN HERE>
  npm install concurrently express body-parser util request http socket.io path http-proxy-middleware request react-router-dom axios socket.io-client react-twitter-embed
  mkdir server
touch server/server.js
  const express = require("express");
const bodyParser = require("body-parser");
const util = require("util");
const request = require("request");
const path = require("path");
const socketIo = require("socket.io");
const http = require("http");

const app = express();
let port = process.env.PORT || 3000;
const post = util.promisify(request.post);
const get = util.promisify(request.get);

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

const server = http.createServer(app);
const io = socketIo(server);

const BEARER_TOKEN = process.env.TWITTER_BEARER_TOKEN;

let timeout = 0;

const streamURL = new URL(
  "https://api.twitter.com/2/tweets/search/stream?tweet.fields=context_annotations&expansions=author_id"
);

const rulesURL = new URL(
  "https://api.twitter.com/2/tweets/search/stream/rules"
);

const errorMessage = {
  title: "Please Wait",
  detail: "Waiting for new Tweets to be posted...",
};

const authMessage = {
  title: "Could not authenticate",
  details: [
    `Please make sure your bearer token is correct. 
      If using Glitch, remix this app and add it to the .env file`,
  ],
  type: "https://developer.twitter.com/en/docs/authentication",
};

const sleep = async (delay) => {
  return new Promise((resolve) => setTimeout(() => resolve(true), delay));
};

app.get("/api/rules", async (req, res) => {
  if (!BEARER_TOKEN) {
    res.status(400).send(authMessage);
  }

  const token = BEARER_TOKEN;
  const requestConfig = {
    url: rulesURL,
    auth: {
      bearer: token,
    },
    json: true,
  };

  try {
    const response = await get(requestConfig);

    if (response.statusCode !== 200) {
      if (response.statusCode === 403) {
        res.status(403).send(response.body);
      } else {
        throw new Error(response.body.error.message);
      }
    }

    res.send(response);
  } catch (e) {
    res.send(e);
  }
});

app.post("/api/rules", async (req, res) => {
  if (!BEARER_TOKEN) {
    res.status(400).send(authMessage);
  }

  const token = BEARER_TOKEN;
  const requestConfig = {
    url: rulesURL,
    auth: {
      bearer: token,
    },
    json: req.body,
  };

  try {
    const response = await post(requestConfig);

    if (response.statusCode === 200 || response.statusCode === 201) {
      res.send(response);
    } else {
      throw new Error(response);
    }
  } catch (e) {
    res.send(e);
  }
});

const streamTweets = (socket, token) => {
  let stream;

  const config = {
    url: streamURL,
    auth: {
      bearer: token,
    },
    timeout: 31000,
  };

  try {
    const stream = request.get(config);

    stream
      .on("data", (data) => {
        try {
          const json = JSON.parse(data);
          if (json.connection_issue) {
            socket.emit("error", json);
            reconnect(stream, socket, token);
          } else {
            if (json.data) {
              socket.emit("tweet", json);
            } else {
              socket.emit("authError", json);
            }
          }
        } catch (e) {
          socket.emit("heartbeat");
        }
      })
      .on("error", (error) => {
        // Connection timed out
        socket.emit("error", errorMessage);
        reconnect(stream, socket, token);
      });
  } catch (e) {
    socket.emit("authError", authMessage);
  }
};

const reconnect = async (stream, socket, token) => {
  timeout++;
  stream.abort();
  await sleep(2 ** timeout * 1000);
  streamTweets(socket, token);
};

io.on("connection", async (socket) => {
  try {
    const token = BEARER_TOKEN;
    io.emit("connect", "Client connected");
    const stream = streamTweets(io, token);
  } catch (e) {
    io.emit("authError", authMessage);
  }
});

console.log("NODE_ENV is", process.env.NODE_ENV);

if (process.env.NODE_ENV === "production") {
  app.use(express.static(path.join(__dirname, "../build")));
  app.get("*", (request, res) => {
    res.sendFile(path.join(__dirname, "../build", "index.html"));
  });
} else {
  port = 3001;
}

server.listen(port, () => console.log(`Listening on port ${port}`));
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/semantic-ui/2.4.1/semantic.min.css" />
import React from "react";
import { BrowserRouter, Route } from "react-router-dom";

import Navbar from "./Navbar";
import TweetFeed from "./TweetFeed";
import RuleList from "./RuleList";

class App extends React.Component {
  render() {
    return (
      <div className="ui container">
        <div className="introduction"></div>

        <h1 className="ui header">
          <div className="content">
            Real Time Tweet Streamer
            <div className="sub header">Powered by Twitter data</div>
          </div>
        </h1>

        <div className="ui container">
          <BrowserRouter>
            <Navbar />
            <Route exact path="/" component={RuleList} />
            <Route exact path="/rules" component={RuleList} />
            <Route exact path="/tweets" component={TweetFeed} />
          </BrowserRouter>
        </div>
      </div>
    );
  }
}

export default App;
import React from "react";
import { NavLink } from "react-router-dom";

const Navbar = () => {
  return (
    <div className="ui two item menu">
      <NavLink to="/tweets" className="item" target="_blank">
        New Tweets
      </NavLink>
      <NavLink to="/rules" className="item" target="_blank">
        Manage Rules
      </NavLink>
    </div>
  );
};

export default Navbar;
import React, { useEffect, useReducer } from "react";
import Tweet from "./Tweet";
import socketIOClient from "socket.io-client";
import ErrorMessage from "./ErrorMessage";
import Spinner from "./Spinner";

const reducer = (state, action) => {
  switch (action.type) {
    case "add_tweet":
      return {
        ...state,
        tweets: [action.payload, ...state.tweets],
        error: null,
        isWaiting: false,
        errors: [],
      };
    case "show_error":
      return { ...state, error: action.payload, isWaiting: false };
    case "add_errors":
      return { ...state, errors: action.payload, isWaiting: false };
    case "update_waiting":
      return { ...state, error: null, isWaiting: true };
    default:
      return state;
  }
};

const TweetFeed = () => {
  const initialState = {
    tweets: [],
    error: {},
    isWaiting: true,
  };

  const [state, dispatch] = useReducer(reducer, initialState);
  const { tweets, error, isWaiting } = state;

  const streamTweets = () => {
    let socket;

    if (process.env.NODE_ENV === "development") {
      socket = socketIOClient("http://localhost:3001/");
    } else {
      socket = socketIOClient("/");
    }

    socket.on("connect", () => {});
    socket.on("tweet", (json) => {
      if (json.data) {
        dispatch({ type: "add_tweet", payload: json });
      }
    });
    socket.on("heartbeat", (data) => {
      dispatch({ type: "update_waiting" });
    });
    socket.on("error", (data) => {
      dispatch({ type: "show_error", payload: data });
    });
    socket.on("authError", (data) => {
      console.log("data =>", data);
      dispatch({ type: "add_errors", payload: [data] });
    });
  };

  const reconnectMessage = () => {
    const message = {
      title: "Reconnecting",
      detail: "Please wait while we reconnect to the stream.",
    };

    if (error && error.detail) {
      return (
        <div>
          <ErrorMessage key={error.title} error={error} styleType="warning" />
          <ErrorMessage
            key={message.title}
            error={message}
            styleType="success"
          />
          <Spinner />
        </div>
      );
    }
  };

  const errorMessage = () => {
    const { errors } = state;

    if (errors && errors.length > 0) {
      return errors.map((error) => (
        <ErrorMessage key={error.title} error={error} styleType="negative" />
      ));
    }
  };

  const waitingMessage = () => {
    const message = {
      title: "Still working",
      detail: "Waiting for new Tweets to be posted",
    };

    if (isWaiting) {
      return (
        <React.Fragment>
          <div>
            <ErrorMessage
              key={message.title}
              error={message}
              styleType="success"
            />
          </div>
          <Spinner />
        </React.Fragment>
      );
    }
  };

  useEffect(() => {
    streamTweets();
  }, []);

  const showTweets = () => {
    if (tweets.length > 0) {
      return (
        <React.Fragment>
          {tweets.map((tweet) => (
            <Tweet key={tweet.data.id} json={tweet} />
          ))}
        </React.Fragment>
      );
    }
  };

  return (
    <div>
      {reconnectMessage()}
      {errorMessage()}
      {waitingMessage()}
      {showTweets()}
    </div>
  );
};

export default TweetFeed;
import React from "react";
import { TwitterTweetEmbed } from "react-twitter-embed";

const Tweet = ({ json }) => {
  const { id } = json.data;

  const options = {
    cards: "hidden",
    align: "center",
    width: "550",
    conversation: "none",
  };

  return <TwitterTweetEmbed options={options} tweetId={id} />;
};

export default Tweet;
import React, { useEffect, useReducer } from "react";
import axios from "axios";
import Rule from "./Rule";
import ErrorMessage from "./ErrorMessage";
import Spinner from "./Spinner";

const reducer = (state, action) => {
  switch (action.type) {
    case "show_rules":
      return { ...state, rules: action.payload, newRule: "" };
    case "add_rule":
      return {
        ...state,
        rules: [...state.rules, ...action.payload],
        newRule: "",
        errors: [],
      };
    case "add_errors":
      return { ...state, rules: state.rules, errors: action.payload };
    case "delete_rule":
      return {
        ...state,
        rules: [...state.rules.filter((rule) => rule.id !== action.payload)],
      };
    case "rule_changed":
      return { ...state, newRule: action.payload };
    case "change_loading_status":
      return { ...state, isLoading: action.payload };
    default:
      return state;
  }
};

const RuleList = () => {
  const initialState = { rules: [], newRule: "", isLoading: false, errors: [] };
  const [state, dispatch] = useReducer(reducer, initialState);
  const exampleRule = "from:twitterdev has:links";
  const ruleMeaning = `This example rule will match Tweets posted by 
     TwtterDev containing links`;
  const operatorsURL =
    "/content/developer-twitter/en/docs/labs/filtered-stream/operators";
  const rulesURL = "/api/rules";

  const createRule = async (e) => {
    e.preventDefault();
    const payload = { add: [{ value: state.newRule }] };

    dispatch({ type: "change_loading_status", payload: true });
    try {
      const response = await axios.post(rulesURL, payload);
      if (response.data.body.errors)
        dispatch({ type: "add_errors", payload: response.data.body.errors });
      else {
        dispatch({ type: "add_rule", payload: response.data.body.data });
      }
      dispatch({ type: "change_loading_status", payload: false });
    } catch (e) {
      dispatch({
        type: "add_errors",
        payload: [{ detail: e.message }],
      });
      dispatch({ type: "change_loading_status", payload: false });
    }
  };

  const deleteRule = async (id) => {
    const payload = { delete: { ids: [id] } };
    dispatch({ type: "change_loading_status", payload: true });
    await axios.post(rulesURL, payload);
    dispatch({ type: "delete_rule", payload: id });
    dispatch({ type: "change_loading_status", payload: false });
  };

  const errors = () => {
    const { errors } = state;

    if (errors && errors.length > 0) {
      return errors.map((error) => (
        <ErrorMessage key={error.title} error={error} styleType="negative" />
      ));
    }
  };

  const rules = () => {
    const { isLoading, rules } = state;

    const message = {
      title: "No rules present",
      details: [
        `There are currently no rules on this stream. Start by adding the rule 
        below.`,
        exampleRule,
        ruleMeaning,
      ],
      type: operatorsURL,
    };

    if (!isLoading) {
      if (rules && rules.length > 0) {
        return rules.map((rule) => (
          <Rule
            key={rule.id}
            data={rule}
            onRuleDelete={(id) => deleteRule(id)}
          />
        ));
      } else {
        return (
          <ErrorMessage
            key={message.title}
            error={message}
            styleType="warning"
          />
        );
      }
    } else {
      return <Spinner />;
    }
  };

  useEffect(() => {
    (async () => {
      dispatch({ type: "change_loading_status", payload: true });

      try {
        const response = await axios.get(rulesURL);
        const { data: payload = [] } = response.data.body;
        dispatch({
          type: "show_rules",
          payload,
        });
      } catch (e) {
        dispatch({ type: "add_errors", payload: [e.response.data] });
      }

      dispatch({ type: "change_loading_status", payload: false });
    })();
  }, []);

  return (
    <div>
      <form onSubmit={(e) => createRule(e)}>
        <div className="ui fluid action input">
          <input
            type="text"
            autoFocus={true}
            value={state.newRule}
            onChange={(e) =>
              dispatch({ type: "rule_changed", payload: e.target.value })
            }
          />
          <button type="submit" className="ui primary button">
            Add Rule
          </button>
        </div>
        {errors()}
        {rules()}
      </form>
    </div>
  );
};

export default RuleList;
import React from "react";

export const Rule = ({ data, onRuleDelete }) => {
  return (
    <div className="ui segment">
      <p>{data.value}</p>
      <div className="ui label">tag: {data.tag}</div>
      <button
        className="ui right floated negative button"
        onClick={() => onRuleDelete(data.id)}
      >
        Delete
      </button>
    </div>
  );
};

export default Rule;
import React from "react";

const ErrorMessage = ({ error, styleType }) => {
  const errorDetails = () => {
    if (error.details) {
      return error.details.map(detail => <p key={detail}>{detail}</p>);
    } else if (error.detail) {
      return <p key={error.detail}>{error.detail}</p>;
    }
  };

  const errorType = () => {
    if (error.type) {
      return (
        <em>
          See
          <a href={error.type} target="_blank" rel="noopener noreferrer">
            {" "}
            Twitter documentation{" "}
          </a>
          for further details.
        </em>
      );
    }
  };

  return (
    <div className={`ui message ${styleType}`}>
      <div className="header">{error.title}</div>
      {errorDetails()}
      {errorType()}
    </div>
  );
};

export default ErrorMessage;
import React from "react";

const Spinner = () => {
  return (
    <div>
      <div className="ui active centered large inline loader"></div>
    </div>
  );
};

export default Spinner;
const { createProxyMiddleware } = require("http-proxy-middleware");

// This proxy redirects requests to /api endpoints to
// the Express server running on port 3001.
module.exports = function (app) {
  app.use(
    ["/api"],
    createProxyMiddleware({
      target: "http://localhost:3001",
    })
  );
};
npm start
