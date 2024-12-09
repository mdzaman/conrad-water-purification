# Here's a step-by-step guide to set up your development environment:

1. Install Required Software:
   - Download and install Visual Studio Code: https://code.visualstudio.com
   - Download and install Node.js: https://nodejs.org (Choose LTS version)
   - Download and install MongoDB Community Edition: https://www.mongodb.com/try/download/community

2. Set Up Database:
```bash
# Start MongoDB
mongod --dbpath /path/to/data/directory
```

3. Create Project Structure:
```bash
mkdir water-purification-project
cd water-purification-project

# Create directories
mkdir backend frontend mobile
```

4. Backend Setup:
```bash
cd backend
npm init -y
npm install express mongoose cors dotenv
```

5. Frontend Setup:
```bash
cd ../frontend
npx create-next-app@latest . --typescript --tailwind --eslint
npm install @radix-ui/react-alert-dialog lucide-react
```

6. Mobile Setup:
```bash
cd ../mobile
npx react-native init WaterPurificationApp
```

7. Environment Configuration:
Create `.env` file in backend folder:
```
MONGODB_URI=mongodb://localhost:27017/water-purification
PORT=3000
```

8. Start Development Servers:
```bash
# Terminal 1 - Backend
cd backend
npm run dev

# Terminal 2 - Frontend
cd frontend
npm run dev

# Terminal 3 - Mobile
cd mobile
npm run start
```

9. Access Your Project:
- Backend API: http://localhost:3000
- Frontend Web: http://localhost:3001
- Mobile: Use Android Emulator or iOS Simulator

