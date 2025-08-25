# Building a Full-Stack MERN Application on AWS EC2

## Overview

This comprehensive guide demonstrates how to build and deploy a complete web application using the MERN technology stack (MongoDB, Express.js, React.js, Node.js) on an Amazon Web Services EC2 instance. The project showcases modern web development practices, cloud deployment strategies, and full-stack application architecture.

## Technology Stack

Our application leverages these core technologies:

- **MongoDB**: A document-oriented NoSQL database that provides flexible schema design and horizontal scalability
- **Express.js**: A minimalist web application framework for Node.js that simplifies API development and middleware integration
- **React.js**: A declarative JavaScript library for building interactive user interfaces with component-based architecture
- **Node.js**: A server-side JavaScript runtime environment that enables non-blocking, event-driven I/O operations

## Development Environment Setup

### AWS EC2 Instance Configuration

1. **Instance Launch**: Deploy a t3.small instance running Ubuntu 24.04 LTS (HVM) in the us-east-2 region for optimal performance and cost-effectiveness.

2. **Security Configuration**: Configure the security group with essential port access:
   - Port 80 (HTTP): Enables web traffic routing
   - Port 22 (SSH): Facilitates secure remote access
   - Ports 5000 and 3000: Reserved for application development and testing

3. **SSH Key Authentication**: Implement key-based authentication for enhanced security by attaching your SSH key during instance creation.

![EC2 Instance Setup](/images/image1.png)

![Security Group Configuration](/images/image2.png)

4. **Remote Connection**: Establish secure connection using your terminal application:

```bash
ssh -i "ssh-key.pem" ubuntu@<your-instance-ip>
```

## Backend Development Phase

### System Preparation

1. **Package Management Update**: Ensure your system has the latest software packages:

```bash
sudo apt update && sudo apt upgrade
```

![System Update](/images/image3.png)

2. **Node.js Installation**: Add the official Node.js repository and install the runtime:

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
```

![Node.js Repository Setup](/images/image4.png)

3. **Runtime Installation**: Install Node.js and npm package manager:

```bash
sudo apt-get install nodejs -y
```

![Node.js Installation](/images/image5.png)

4. **Verification**: Confirm successful installation by checking version numbers:

```bash
node -v
npm -v
```

### Project Initialization

1. **Directory Structure**: Create and navigate to your project workspace:

```bash
mkdir Todo && ls && cd Todo
npm init
```

![Package.json Initialization](/images/image6.png)

The `package.json` file serves as your project's configuration manifest, documenting dependencies, scripts, and metadata.

### Express.js Framework Setup

Express.js provides a robust foundation for building web applications and APIs with minimal boilerplate code.

1. **Framework Installation**:

```bash
npm install express
```

2. **Application Entry Point**: Create the main server file:

```bash
touch index.js && ls
```

3. **Environment Management**: Install dotenv for secure configuration handling:

```bash
npm install dotenv
```

![Entry Point Creation](/images/image7.png)

4. **Basic Server Implementation**: Configure your Express application:

```bash
sudo nano index.js
```
**Add the configuration below**:

```javascript
const express = require('express');
require('dotenv').config();

const app = express();

const port = process.env.PORT || 5000;

app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  next();
});

app.use((req, res, next) => {
  res.send('Welcome to Express');
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

5. **Server Launch**: Start your development server:

  ***Remember to edit Inbound rules by adding Custom TCP port 5000***

![Inbound Rules:Port 5000](/images/image8.png)

```bash
node index.js
```

![Server Startup](/images/image9.png)

Access your application at `http://your-instance-ip:5000` to verify the server is operational.

![Server Verification](/images/image10.png)

---



## API Architecture Implementation

### Route Structure Design

Implement RESTful API endpoints to handle CRUD operations for your todo application:

1. **Route Directory Creation**:

```bash
mkdir routes && cd routes && touch api.js
```

![Route Structure](/images/image11.png)

2. **API Endpoint Definition**:

```javascript
const express = require('express');
const router = express.Router();

router.get('/todos', (req, res, next) => {
  // Retrieve all todo items
});

router.post('/todos', (req, res, next) => {
  // Create new todo item
});

router.delete('/todos/:id', (req, res, next) => {
  // Remove specific todo item
});

module.exports = router;
```
---

## Schema Definition: Database Setup and Model Creation

### Step 1: Navigate to Todo Directory and Install Mongoose

```bash
cd ..
npm install mongoose
```

### Step 2: Create Models Directory and Todo Model

```bash
mkdir models && cd models && touch todo.js
```
Now that we have created the models directory and todo.js file, let's define the schema:

```javascript
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const TodoSchema = new Schema({
  action: {
    type: String,
    required: [true, 'The todo text field is required']
  }
});

const Todo = mongoose.model('todo', TodoSchema);

module.exports = Todo;
```

### Route Integration

Update your API routes to utilize the data model:

```javascript
const express = require('express');
const router = express.Router();
const Todo = require('../models/todo');

router.get('/todos', (req, res, next) => {
  Todo.find({}, 'action')
    .then(data => res.json(data))
    .catch(next);
});

router.post('/todos', (req, res, next) => {
  if (req.body.action) {
    Todo.create(req.body)
      .then(data => res.json(data))
      .catch(next);
  } else {
    res.json({
      error: "The input field is empty"
    });
  }
});

router.delete('/todos/:id', (req, res, next) => {
  Todo.findOneAndDelete({"_id": req.params.id})
    .then(data => res.json(data))
    .catch(next);
});

module.exports = router;
```

## Database Configuration

### MongoDB Atlas Setup

1. **Cloud Database Creation**: 
   - Register for a MongoDB Atlas account
   - Create a new cluster in the AWS cloud infrastructure
   - Select the Paris (eu-west-3) region for optimal performance
   - Configure network access to allow connections from any IP address

<!-- ![MongoDB Atlas Setup](/images/image12b.png) -->

![Cluster Configuration](/images/image12.png)

2. **Database and Collection Setup**:
   - Create a database named `todo_db`
   - Initialize a collection named `todos`

![Database Creation](/images/image13.png)

3. **Environment Configuration**: Create a `.env` file for secure credential management:

```bash
touch .env && vi .env
```

Add your MongoDB connection string:

```bash
DB = 'mongodb+srv://<username>:<password>@<network-address>/<dbname>?retryWrites=true&w=majority'
```

4. **Server Integration**: Update your main server file to include database connectivity:

```javascript
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const routes = require('./routes/api');
const path = require('path');
require('dotenv').config();

const app = express();

const port = process.env.PORT || 5000;

mongoose.connect(process.env.DB, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log(`Database connected successfully`))
  .catch(err => console.log(err));

mongoose.Promise = global.Promise;

app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  next();
});

app.use(bodyParser.json());

app.use('/api', routes);

app.use((err, req, res, next) => {
  console.log(err);
  next();
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

![Server Integration](/images/mern22.jpg)

5. **Application Launch**: Start your backend server:

```bash
node index.js
```

![Backend Launch](/images/mern23.jpg)

## API Testing and Validation

### Postman Integration Testing

1. **Request Configuration**: Configure Postman to test your API endpoints at `http://your-instance-ip:5000/api/todos`

![Postman Setup](/images/mern24.jpg)

2. **GET Request Testing**: Retrieve all todo items from the database

![GET Request](/images/mern25.jpg)

3. **Database Verification**: Confirm data persistence in MongoDB Atlas

![Database Verification](/images/mern26.jpg)

4. **DELETE Request Testing**: Remove completed tasks from the system

![DELETE Request](/images/mern27.jpg)

5. **Post-Deletion Verification**: Validate successful data removal

![Post-Deletion Check](/images/mern28.jpg)

## Frontend Development

### React Application Setup

1. **React Project Creation**: Initialize a new React application in your project directory:

```bash
npx create-react-app client
```

![React App Creation](/images/mern29.jpg)

### Development Environment Configuration

1. **Concurrent Execution Setup**: Install tools for simultaneous backend and frontend development:

```bash
npm install concurrently --save-dev
```

![Concurrent Setup](/images/mern30.jpg)

2. **Script Configuration**: Update your package.json scripts for streamlined development:

```json
"scripts": {
  "start": "node index.js",
  "start-watch": "nodemon index.js",
  "dev": "concurrently \"npm run start-watch\" \"cd client && npm start\""
}
```

![Script Configuration](/images/mern31.jpg)

### Proxy Configuration

1. **Client Directory Navigation**:

```bash
cd client
```

2. **Package.json Modification**: Add proxy configuration for seamless API communication:

```bash
vi package.json
```

Add the proxy line:

```json
"proxy": "http://localhost:5000"
```

![Proxy Setup](/images/mern32.jpg)

3. **Development Server Launch**: Start both backend and frontend simultaneously:

```bash
npm run dev
```

![Development Server](/images/mern33.jpg)

## React Component Architecture

### Component Structure Design

1. **Component Directory Creation**:

```bash
cd client/src
mkdir components
cd components
```

2. **Component File Initialization**:

```bash
touch Input.js ListTodo.js Todo.js
```

### Input Component Implementation

Create a component for task input functionality:

```javascript
import React, { Component } from 'react';
import axios from 'axios';

class Input extends Component {
  state = {
    action: ""
  }

  handleChange = (event) => {
    this.setState({ action: event.target.value });
  }

  addTodo = () => {
    const task = { action: this.state.action };

    if (task.action && task.action.length > 0) {
      axios.post('/api/todos', task)
        .then(res => {
          if (res.data) {
            this.props.getTodos();
            this.setState({ action: "" });
          }
        })
        .catch(err => console.log(err));
    } else {
      console.log('Input field required');
    }
  }

  render() {
    let { action } = this.state;
    return (
      <div>
        <input type="text" onChange={this.handleChange} value={action} />
        <button onClick={this.addTodo}>add todo</button>
      </div>
    );
  }
}

export default Input;
```

### HTTP Client Installation

Install Axios for API communication:

```bash
cd ../..
npm install axios
```

![Axios Installation](/images/mern34.jpg)

### List Component Implementation

Create a component for displaying todo items:

```javascript
import React from 'react';

const ListTodo = ({ todos, deleteTodo }) => {
  return (
    <ul>
      {
        todos && todos.length > 0 ? (
          todos.map(todo => {
            return (
              <li key={todo._id} onClick={() => deleteTodo(todo._id)}>
                {todo.action}
              </li>
            );
          })
        ) : (
          <li>No todo(s) left</li>
        )
      }
    </ul>
  );
}

export default ListTodo;
```

### Main Todo Component

Implement the primary component that orchestrates the application:

```javascript
import React, { Component } from 'react';
import axios from 'axios';

import Input from './Input';
import ListTodo from './ListTodo';

class Todo extends Component {
  state = {
    todos: []
  }

  componentDidMount() {
    this.getTodos();
  }

  getTodos = () => {
    axios.get('/api/todos')
      .then(res => {
        if (res.data) {
          this.setState({
            todos: res.data
          });
        }
      })
      .catch(err => console.log(err));
  }

  deleteTodo = (id) => {
    axios.delete(`/api/todos/${id}`)
      .then(res => {
        if (res.data) {
          this.getTodos();
        }
      })
      .catch(err => console.log(err));
  }

  render() {
    let { todos } = this.state;
    return (
      <div>
        <h1>My Todo(s)</h1>
        <Input getTodos={this.getTodos} />
        <ListTodo todos={todos} deleteTodo={this.deleteTodo} />
      </div>
    );
  }
}

export default Todo;
```

![Main Component](/images/mern35.jpg)

### Application Integration

1. **App.js Update**: Modify the main application file to include your Todo component:

```javascript
import React from 'react';
import Todo from './components/Todo';
import './App.css';

const App = () => {
  return (
    <div className="App">
      <Todo />
    </div>
  );
}

export default App;
```

### Styling Implementation

1. **App.css Styling**: Apply modern, responsive design patterns:

```css
.App {
  text-align: center;
  font-size: calc(10px + 2vmin);
  width: 60%;
  margin-left: auto;
  margin-right: auto;
}

input {
  height: 40px;
  width: 50%;
  border: none;
  border-bottom: 2px #101113 solid;
  background: none;
  font-size: 1.5rem;
  color: #787a80;
}

input:focus {
  outline: none;
}

button {
  width: 25%;
  height: 45px;
  border: none;
  margin-left: 10px;
  font-size: 25px;
  background: #101113;
  border-radius: 5px;
  color: #787a80;
  cursor: pointer;
}

button:focus {
  outline: none;
}

ul {
  list-style: none;
  text-align: left;
  padding: 15px;
  background: #171a1f;
  border-radius: 5px;
}

li {
  padding: 15px;
  font-size: 1.5rem;
  margin-bottom: 15px;
  background: #282c34;
  border-radius: 5px;
  overflow-wrap: break-word;
  cursor: pointer;
}

@media only screen and (min-width: 300px) {
  .App {
    width: 80%;
  }

  input {
    width: 100%
  }

  button {
    width: 100%;
    margin-top: 15px;
    margin-left: 0;
  }
}

@media only screen and (min-width: 640px) {
  .App {
    width: 60%;
  }

  input {
    width: 50%;
  }

  button {
    width: 30%;
    margin-left: 10px;
    margin-top: 0;
  }
}
```

![Styling Implementation](/images/mern37.jpg)

2. **Global Styles**: Update index.css for consistent theming:

```css
body {
  margin: 0;
  padding: 0;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Roboto", "Oxygen", "Ubuntu", "Cantarell", "Fira Sans", "Droid Sans", "Helvetica Neue", sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  box-sizing: border-box;
  background-color: #282c34;
  color: #787a80;
}

code {
  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New", monospace;
}
```

## Application Deployment and Testing

### Full-Stack Application Launch

1. **Concurrent Execution**: Start both backend and frontend services:

```bash
npm run dev
```

![Full Application Launch](/images/mern38.jpg)

2. **Browser Access**: Navigate to your EC2 instance's public IP address on port 3000 to access your fully functional todo application.

![Application Interface](/images/mern39.jpg)

## Project Summary

This comprehensive guide demonstrates the complete development lifecycle of a modern full-stack web application. The project encompasses:

- **Cloud Infrastructure Setup**: AWS EC2 instance configuration and security management
- **Backend Development**: Express.js API development with MongoDB integration
- **Frontend Implementation**: React.js component architecture and state management
- **Database Design**: MongoDB Atlas cloud database configuration
- **Development Workflow**: Concurrent development environment setup
- **Application Deployment**: Production-ready application deployment on cloud infrastructure

The resulting application showcases modern web development best practices, cloud deployment strategies, and full-stack JavaScript development methodologies.
