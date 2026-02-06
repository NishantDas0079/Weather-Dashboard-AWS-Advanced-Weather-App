# 🚀 Installation & Setup
```
# Using PuTTY (Windows) or SSH (Linux/Mac)
ssh -i "your-key.pem" ubuntu@your-ec2-public-ip
```

# Initial Server Setup
```
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Node.js 18.x
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version
npm --version

# Install PM2 for process management
sudo npm install -g pm2
```

# Clone and Setup Project
```
# Create project directory
mkdir weather-dashboard
cd weather-dashboard

# Initialize Node.js project
npm init -y

# Install dependencies
npm install express axios dotenv
```

# Project Files Setup
# package.json
```json
{
  "name": "weather-dashboard",
  "version": "1.0.0",
  "description": "Real-time Weather Dashboard",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "pm2:start": "pm2 start server.js --name weather-dashboard",
    "pm2:stop": "pm2 stop weather-dashboard",
    "pm2:restart": "pm2 restart weather-dashboard",
    "pm2:logs": "pm2 logs weather-dashboard"
  },
  "dependencies": {
    "express": "^4.18.2",
    "axios": "^1.5.0",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

#.env configuration
```
# OpenWeatherMap API Key (Get from: https://openweathermap.org/api)
WEATHER_API_KEY=your_api_key_here

# Server Configuration
PORT=80
NODE_ENV=production

# API Settings
API_TIMEOUT=5000
API_MAX_RETRIES=3
```

# Option A: Real Weather Data (Recommended)
Get Free API Key:

Visit: https://openweathermap.org/api

Sign up for free account

Verify email

Generate API key from dashboard

Copy your key: 8c672cb9aa593491e7f2570fe5599a89 (example)

Configure Environment:
```
# Edit .env file
nano .env

# Add your API key
WEATHER_API_KEY=8c672cb9aa593491e7f2570fe5599a89
PORT=80
```

# Option B: Demo Mode (No API required)
```
# For demo mode, leave API key empty or use demo value
WEATHER_API_KEY=demo_mode
PORT=80
```

# 🚀 Deployment on AWS EC2

# Step 1: Security Group Configuration
Configure EC2 Security Group to allow:

SSH (Port 22) - For server access

HTTP (Port 80) - For web traffic

HTTPS (Port 443) - Optional for SSL

# Step 2: Transfer Files to EC2
```
# Using SCP (from local machine)
scp -i "your-key.pem" -r ./weather-dashboard ubuntu@your-ec2-ip:/home/ubuntu/

# Or clone from GitHub
git clone https://github.com/yourusername/weather-dashboard.git
cd weather-dashboard
```

# Step 3 : Install and Run
```
# Install dependencies
npm install

# Test the application
npm start

# For production with PM2
pm2 start server.js --name "weather-dashboard"
pm2 save
pm2 startup
```

# Step 4: Configure Port 80 Access
```
# Run Node.js on port 80 (requires sudo)
sudo PORT=80 node server.js

# Or with PM2 (as root)
sudo pm2 start server.js --name "weather-dashboard" -- --port 80
```

# 📡 API Documentation
Available Endpoints
1. GET / - Main Application
```
GET http://your-ec2-ip/
```

2. GET /api/weather - Weather Data
```
GET http://your-ec2-ip/api/weather?city=Mumbai
```

Response Example :
```json
{
  "city": "Mumbai",
  "country": "IN",
  "temperature": 28,
  "feels_like": 31,
  "humidity": 75,
  "description": "Partly cloudy",
  "icon": "04d",
  "wind_speed": 12,
  "visibility": 10,
  "sunrise": "06:45:00",
  "sunset": "18:30:00",
  "real_data": true,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

3. GET /api/status - API Status
```
GET http://your-ec2-ip/api/status
```

Response :
```json
{
  "status": "online",
  "api_key": "Configured",
  "mode": "REAL DATA",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

4. GET /health - Health Check
```
GET http://your-ec2-ip/health
```

Response :
```json
{
  "status": "healthy",
  "service": "Weather Dashboard",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

# Starting the Application
```
# Development mode (with auto-restart)
npm run dev

# Production mode
npm start

# Production with PM2 (recommended)
npm run pm2:start

# View logs
npm run pm2:logs
```

# Accessing the Dashboard
```
curl ifconfig.me

http://YOUR_EC2_PUBLIC_IP
```

Default cities available:

Mumbai, Delhi, Bangalore

London, New York, Tokyo

Dubai, Paris, Sydney

Any other city worldwide

# Using the Interface
Search City: Type any city name in search box

Quick Select: Click on popular city buttons

Real-time Data: Check "Real-time API Data" badge

Weather Details: View temperature, humidity, wind, etc.

Auto-refresh: Data updates every 5 minutes

# Common Issues and Solutions
# Issue 1: "Invalid API Key" Error
```
# Test your API key
curl "https://api.openweathermap.org/data/2.5/weather?q=Mumbai&appid=YOUR_API_KEY"

# Solutions:
# 1. Wait 10 minutes after signup (activation delay)
# 2. Verify email address
# 3. Generate new API key
# 4. Use demo mode temporarily
```

# Issue 2: Port 80 Already in Use
```
# Check what's using port 80
sudo lsof -i :80

# Kill the process or use different port
sudo kill -9 <PID>
# Or change PORT in .env to 3000
```

# Issue 3: Application Not Starting
```
# Check Node.js version
node --version

# Check dependencies
npm list

# Check logs
pm2 logs weather-dashboard

# Common fix: Reinstall dependencies
rm -rf node_modules package-lock.json
npm install
```

# Logs and Monitoring
```
# View PM2 logs
pm2 logs weather-dashboard

# View system logs
sudo tail -f /var/log/syslog

# Monitor resources
htop
pm2 monit
```

# 🚀 Deployment Commands Cheatsheet
```
# 1. Initial Setup
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2

# 2. Project Setup
git clone https://github.com/yourusername/weather-dashboard.git
cd weather-dashboard
npm install

# 3. Configuration
cp .env.example .env
nano .env  # Add your API key

# 4. Start Application
pm2 start server.js --name "weather-dashboard"
pm2 save
pm2 startup

# 5. Monitor
pm2 status
pm2 logs
```

# 📊 Performance Metrics
Page Load Time: < 2 seconds

API Response Time: < 1 second

Uptime: 99.9% (with PM2)

Concurrent Users: 100+ (on t2.micro)

Data Accuracy: Real-time (OpenWeatherMap)

# Planned Features
User Authentication - Save favorite cities

Weather Forecast - 5-day forecast view

Location Detection - Auto-detect user location

Weather Maps - Interactive weather maps

Multiple Languages - Internationalization

Dark/Light Mode - Theme switching

Weather Alerts - Severe weather notifications

Historical Data - Weather trends and analytics
