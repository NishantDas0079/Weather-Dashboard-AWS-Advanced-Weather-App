# 🔧 Configuration
Get a free API key from OpenWeatherMap

Create a .env file:

```
API_KEY=your_api_key_here
LOG_LEVEL=INFO
SAVE_REPORTS=true
```

Run the application

# 🐳 Docker Support
```
# Build the image
docker build -t weather-app .

# Run the container
docker run -it --env-file .env weather-app
```

#  AWS EC2 Deployment
```
# Run setup script on EC2
chmod +x scripts/setup.sh
./scripts/setup.sh
```

# Sample Output
```
============================================================
🌍 CURRENT WEATHER: LONDON, GB ☁️
============================================================
📊 Temperature: 15°C (Feels like: 14°C)
💧 Humidity: 75%
📈 Pressure: 1013 hPa
🌤️ Condition: Cloudy
💨 Wind: 5.2 m/s, Direction: 270°
👁️ Visibility: 10000 meters
☁️ Cloudiness: 90%
🌅 Sunrise: 06:45:00
🌇 Sunset: 19:30:00
📍 Coordinates: Lat 51.51, Lon -0.13
============================================================
```

# requirements.txt
```
requests==2.31.0
python-dotenv==1.0.0
pandas==2.1.4
matplotlib==3.8.2
pytest==7.4.3
colorama==0.4.6
tabulate==0.9.0
rich==13.7.0
click==8.1.7

# Optional for advanced features
boto3==1.34.0      # AWS SDK
flask==3.0.0       # Web interface
sqlalchemy==2.0.23 # Database
```

# .env.example
```
# OpenWeatherMap API Configuration
API_KEY=your_api_key_here

# Application Settings
LOG_LEVEL=INFO
SAVE_REPORTS=true
TEMPERATURE_UNIT=metric  # metric, imperial, or standard
LANGUAGE=en

# Display Settings
SHOW_EMOJIS=true
SHOW_CHARTS=true
AUTO_SAVE=false

# AWS Configuration (Optional)
AWS_REGION=us-east-1
S3_BUCKET=your-weather-reports-bucket
CLOUDWATCH_LOG_GROUP=weather-app-logs

# Email Alerts (Optional)
EMAIL_ENABLED=false
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
EMAIL_USER=your_email@gmail.com
EMAIL_PASSWORD=your_app_password

# Database (Optional)
DB_ENABLED=false
DB_PATH=weather_data.db
```

# src/main.py
```
#!/usr/bin/env python3
"""
Advanced Weather Application - Main Entry Point
"""

import os
import sys
from pathlib import Path

# Add src directory to Python path
sys.path.insert(0, str(Path(__file__).parent))

from weather_app import WeatherApp
from utils import setup_logging, check_dependencies
from config import load_config

def main():
    """Main application entry point"""
    
    print("\n" + "="*60)
    print("🌤️  ADVANCED WEATHER FORECAST APPLICATION v2.0")
    print("="*60)
    
    # Check dependencies
    if not check_dependencies():
        print("\n❌ Missing dependencies. Please install requirements:")
        print("   pip install -r requirements.txt")
        sys.exit(1)
    
    # Setup logging
    logger = setup_logging()
    logger.info("Starting Advanced Weather Application")
    
    # Load configuration
    config = load_config()
    
    try:
        # Initialize and run the application
        app = WeatherApp(config)
        app.run()
        
    except KeyboardInterrupt:
        print("\n\n👋 Application terminated by user")
        logger.info("Application terminated by user")
    except Exception as e:
        print(f"\n❌ Fatal error: {e}")
        logger.error(f"Fatal error: {e}", exc_info=True)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

# src/weather_app.py
```
"""
Main Weather Application Class
"""

import requests
import json
import os
from datetime import datetime, timedelta
from typing import Dict, List, Optional, Tuple
from colorama import init, Fore, Style
from tabulate import tabulate
import sys

from utils import (
    clear_screen, 
    format_temperature, 
    get_weather_icon,
    create_ascii_chart,
    save_report,
    validate_city_name
)
from config import Config

init(autoreset=True)  # Initialize colorama

class WeatherApp:
    """Main Weather Application Class"""
    
    def __init__(self, config: Config):
        self.config = config
        self.api_key = config.api_key
        self.base_url = "http://api.openweathermap.org/data/2.5"
        self.history: List[Dict] = []
        self.current_city: Optional[str] = None
        
        if not self.api_key or self.api_key == "YOUR_API_KEY_HERE":
            self._setup_api_key()
    
    def _setup_api_key(self):
        """Setup API key interactively"""
        print(f"\n{Fore.YELLOW}⚠️  API Key Configuration Required{Style.RESET_ALL}")
        print(f"{Fore.CYAN}="*50)
        print("To use this application, you need an OpenWeatherMap API key.")
        print("1. Visit: https://openweathermap.org/api")
        print("2. Sign up for a free account")
        print("3. Get your API key from the dashboard")
        print(f"{Fore.CYAN}="*50)
        
        api_key = input(f"\n{Fore.GREEN}Enter your API key: {Style.RESET_ALL}").strip()
        
        if api_key:
            self.api_key = api_key
            # Save to .env file
            with open('.env', 'w') as f:
                f.write(f"API_KEY={api_key}\n")
            print(f"{Fore.GREEN}✅ API key saved to .env file{Style.RESET_ALL}")
        else:
            print(f"{Fore.YELLOW}⚠️  Running in demo mode with limited features{Style.RESET_ALL}")
            print(f"{Fore.YELLOW}   For full functionality, please set API_KEY in .env file{Style.RESET_ALL}")
    
    def _make_api_request(self, endpoint: str, params: Dict) -> Optional[Dict]:
        """Make API request with error handling"""
        try:
            url = f"{self.base_url}/{endpoint}"
            params['appid'] = self.api_key
            params['units'] = self.config.temperature_unit
            
            response = requests.get(url, params=params, timeout=10)
            
            if response.status_code == 200:
                return response.json()
            elif response.status_code == 401:
                print(f"{Fore.RED}❌ Invalid API key. Please check your .env file{Style.RESET_ALL}")
                return None
            elif response.status_code == 404:
                print(f"{Fore.RED}❌ City not found. Please check spelling{Style.RESET_ALL}")
                return None
            elif response.status_code == 429:
                print(f"{Fore.YELLOW}⚠️  Too many requests. Please wait...{Style.RESET_ALL}")
                return None
            else:
                error_msg = response.json().get('message', 'Unknown error')
                print(f"{Fore.RED}❌ Error {response.status_code}: {error_msg}{Style.RESET_ALL}")
                return None
                
        except requests.exceptions.ConnectionError:
            print(f"{Fore.RED}❌ Network error: Cannot connect to weather service{Style.RESET_ALL}")
            return None
        except requests.exceptions.Timeout:
            print(f"{Fore.RED}❌ Request timeout: Service not responding{Style.RESET_ALL}")
            return None
        except Exception as e:
            print(f"{Fore.RED}❌ Unexpected error: {str(e)}{Style.RESET_ALL}")
            return None
    
    def get_current_weather(self, city: str) -> Optional[Dict]:
        """Get current weather for a city"""
        print(f"{Fore.CYAN}🔍 Fetching current weather for {city}...{Style.RESET_ALL}")
        
        params = {
            'q': city,
            'lang': self.config.language
        }
        
        data = self._make_api_request('weather', params)
        
        if data:
            self.current_city = city
            self.history.append({
                'city': city,
                'timestamp': datetime.now(),
                'data': data
            })
            
            # Keep only last 10 searches
            if len(self.history) > 10:
                self.history = self.history[-10:]
        
        return data
    
    def get_forecast(self, city: str) -> Optional[Dict]:
        """Get 5-day weather forecast"""
        print(f"{Fore.CYAN}🔍 Fetching 5-day forecast for {city}...{Style.RESET_ALL}")
        
        params = {
            'q': city,
            'cnt': 40,  # 5 days * 8 intervals per day
            'lang': self.config.language
        }
        
        data = self._make_api_request('forecast', params)
        
        if data:
            self.current_city = city
        
        return data
    
    def display_current_weather(self, data: Dict):
        """Display formatted current weather"""
        if not data:
            return
        
        city = data.get('name', 'Unknown')
        country = data.get('sys', {}).get('country', '')
        weather = data.get('weather', [{}])[0]
        
        # Format output
        output = [
            f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}",
            f"{Fore.YELLOW}🌍 CURRENT WEATHER: {city.upper()}, {country} {get_weather_icon(weather.get('id', 800))}{Style.RESET_ALL}",
            f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}",
            "",
            f"{Fore.WHITE}📊 {Fore.GREEN}Temperature:{Style.RESET_ALL} {format_temperature(data['main']['temp'])} "
            f"(Feels like: {format_temperature(data['main']['feels_like'])})",
            "",
            f"{Fore.WHITE}💧 {Fore.GREEN}Humidity:{Style.RESET_ALL} {data['main']['humidity']}%",
            f"{Fore.WHITE}📈 {Fore.GREEN}Pressure:{Style.RESET_ALL} {data['main']['pressure']} hPa",
            f"{Fore.WHITE}🌤️  {Fore.GREEN}Condition:{Style.RESET_ALL} {weather.get('description', '').title()}",
            f"{Fore.WHITE}💨 {Fore.GREEN}Wind:{Style.RESET_ALL} {data['wind']['speed']} m/s, "
            f"Direction: {data['wind'].get('deg', 'N/A')}°",
            f"{Fore.WHITE}👁️  {Fore.GREEN}Visibility:{Style.RESET_ALL} "
            f"{data.get('visibility', 'N/A')} meters",
            f"{Fore.WHITE}☁️  {Fore.GREEN}Cloudiness:{Style.RESET_ALL} {data['clouds']['all']}%",
            "",
        ]
        
        # Add sunrise/sunset if available
        if 'sys' in data and 'sunrise' in data['sys']:
            sunrise = datetime.fromtimestamp(data['sys']['sunrise']).strftime('%H:%M:%S')
            sunset = datetime.fromtimestamp(data['sys']['sunset']).strftime('%H:%M:%S')
            output.extend([
                f"{Fore.WHITE}🌅 {Fore.GREEN}Sunrise:{Style.RESET_ALL} {sunrise}",
                f"{Fore.WHITE}🌇 {Fore.GREEN}Sunset:{Style.RESET_ALL} {sunset}",
                ""
            ])
        
        # Add coordinates
        coord = data.get('coord', {})
        output.append(f"{Fore.WHITE}📍 {Fore.GREEN}Coordinates:{Style.RESET_ALL} "
                     f"Lat {coord.get('lat', 'N/A')}, Lon {coord.get('lon', 'N/A')}")
        
        output.append(f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}")
        
        print("\n".join(output))
    
    def display_forecast(self, data: Dict):
        """Display formatted 5-day forecast"""
        if not data:
            return
        
        city = data.get('city', {}).get('name', 'Unknown')
        country = data.get('city', {}).get('country', '')
        
        print(f"\n{Fore.CYAN}{'='*60}{Style.RESET_ALL}")
        print(f"{Fore.YELLOW}📅 5-DAY FORECAST: {city.upper()}, {country}{Style.RESET_ALL}")
        print(f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}")
        
        # Group forecasts by day
        daily_data = {}
        for item in data.get('list', []):
            date = item['dt_txt'].split()[0]
            if date not in daily_data:
                daily_data[date] = []
            daily_data[date].append(item)
        
        # Display each day
        table_data = []
        for date in sorted(daily_data.keys())[:5]:
            day_forecasts = daily_data[date]
            date_obj = datetime.strptime(date, '%Y-%m-%d')
            day_name = date_obj.strftime('%A')
            
            # Calculate statistics
            temps = [f['main']['temp'] for f in day_forecasts]
            max_temp = max(temps)
            min_temp = min(temps)
            avg_temp = sum(temps) / len(temps)
            
            # Get most common weather
            conditions = [f['weather'][0]['main'] for f in day_forecasts]
            common_condition = max(set(conditions), key=conditions.count)
            condition_desc = day_forecasts[0]['weather'][0]['description'].title()
            weather_id = day_forecasts[0]['weather'][0]['id']
            
            table_data.append([
                date,
                day_name,
                f"{format_temperature(avg_temp)}",
                f"{format_temperature(max_temp)}",
                f"{format_temperature(min_temp)}",
                f"{get_weather_icon(weather_id)} {common_condition}",
                condition_desc
            ])
        
        # Display table
        headers = [
            f"{Fore.WHITE}Date{Style.RESET_ALL}",
            f"{Fore.WHITE}Day{Style.RESET_ALL}",
            f"{Fore.WHITE}Avg Temp{Style.RESET_ALL}",
            f"{Fore.WHITE}High{Style.RESET_ALL}",
            f"{Fore.WHITE}Low{Style.RESET_ALL}",
            f"{Fore.WHITE}Condition{Style.RESET_ALL}",
            f"{Fore.WHITE}Description{Style.RESET_ALL}"
        ]
        
        print(tabulate(table_data, headers=headers, tablefmt="grid"))
        
        # City info
        city_info = data.get('city', {})
        print(f"\n{Fore.CYAN}{'─'*60}{Style.RESET_ALL}")
        print(f"{Fore.WHITE}🏙️  City Information:{Style.RESET_ALL}")
        print(f"   {Fore.WHITE}├── {Fore.GREEN}Population:{Style.RESET_ALL} {city_info.get('population', 'N/A')}")
        print(f"   {Fore.WHITE}├── {Fore.GREEN}Timezone:{Style.RESET_ALL} UTC{city_info.get('timezone', 0)//3600:+d}")
        print(f"   {Fore.WHITE}└── {Fore.GREEN}Coordinates:{Style.RESET_ALL} "
              f"{city_info.get('coord', {}).get('lat', 'N/A')}, "
              f"{city_info.get('coord', {}).get('lon', 'N/A')}")
        
        print(f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}")
    
    def show_temperature_chart(self, forecast_data: Dict):
        """Display ASCII temperature chart"""
        if not forecast_data or not self.config.show_charts:
            return
        
        create_ascii_chart(forecast_data)
    
    def show_history(self):
        """Display search history"""
        if not self.history:
            print(f"\n{Fore.YELLOW}No search history available{Style.RESET_ALL}")
            return
        
        print(f"\n{Fore.CYAN}{'='*60}{Style.RESET_ALL}")
        print(f"{Fore.YELLOW}📜 SEARCH HISTORY (Last 10 cities){Style.RESET_ALL}")
        print(f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}")
        
        table_data = []
        for i, entry in enumerate(reversed(self.history), 1):
            city = entry['city']
            timestamp = entry['timestamp'].strftime('%Y-%m-%d %H:%M:%S')
            temp = entry['data'].get('main', {}).get('temp', 'N/A')
            condition = entry['data'].get('weather', [{}])[0].get('description', 'N/A').title()
            
            table_data.append([
                i,
                city,
                format_temperature(temp) if temp != 'N/A' else 'N/A',
                condition,
                timestamp
            ])
        
        headers = [
            f"{Fore.WHITE}#{Style.RESET_ALL}",
            f"{Fore.WHITE}City{Style.RESET_ALL}",
            f"{Fore.WHITE}Temperature{Style.RESET_ALL}",
            f"{Fore.WHITE}Condition{Style.RESET_ALL}",
            f"{Fore.WHITE}Time{Style.RESET_ALL}"
        ]
        
        print(tabulate(table_data, headers=headers, tablefmt="grid"))
        print(f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}")
    
    def run(self):
        """Main application loop"""
        while True:
            clear_screen()
            self._display_menu()
            
            choice = input(f"\n{Fore.GREEN}Enter your choice (1-8): {Style.RESET_ALL}").strip()
            
            if choice == '8':
                print(f"\n{Fore.YELLOW}👋 Thank you for using Advanced Weather App! Goodbye!{Style.RESET_ALL}")
                break
            
            self._handle_choice(choice)
    
    def _display_menu(self):
        """Display main menu"""
        menu = [
            f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}",
            f"{Fore.YELLOW}🌤️  ADVANCED WEATHER APP v2.0{Style.RESET_ALL}",
            f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}",
            "",
            f"{Fore.WHITE}1. {Fore.GREEN}🌍 Get Current Weather{Style.RESET_ALL}",
            f"{Fore.WHITE}2. {Fore.GREEN}📅 Get 5-Day Forecast{Style.RESET_ALL}",
            f"{Fore.WHITE}3. {Fore.GREEN}🌟 Get Current + Forecast + Chart{Style.RESET_ALL}",
            f"{Fore.WHITE}4. {Fore.GREEN}📊 Show Temperature Chart{Style.RESET_ALL}",
            f"{Fore.WHITE}5. {Fore.GREEN}📜 View Search History{Style.RESET_ALL}",
            f"{Fore.WHITE}6. {Fore.GREEN}💾 Save Last Report to File{Style.RESET_ALL}",
            f"{Fore.WHITE}7. {Fore.GREEN}❓ Help / Instructions{Style.RESET_ALL}",
            f"{Fore.WHITE}8. {Fore.RED}🚪 Exit{Style.RESET_ALL}",
            "",
            f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}"
        ]
        
        print("\n".join(menu))
    
    def _handle_choice(self, choice: str):
        """Handle user menu choice"""
        if choice == '1':
            self._handle_current_weather()
        elif choice == '2':
            self._handle_forecast()
        elif choice == '3':
            self._handle_complete_report()
        elif choice == '4':
            self._handle_chart()
        elif choice == '5':
            self._handle_history()
        elif choice == '6':
            self._handle_save()
        elif choice == '7':
            self._handle_help()
        else:
            print(f"\n{Fore.RED}❌ Invalid choice. Please enter 1-8{Style.RESET_ALL}")
            input(f"{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
    
    def _handle_current_weather(self):
        """Handle current weather request"""
        city = input(f"\n{Fore.GREEN}Enter city name: {Style.RESET_ALL}").strip()
        if not city:
            print(f"{Fore.RED}❌ City name cannot be empty!{Style.RESET_ALL}")
            input(f"{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
            return
        
        data = self.get_current_weather(city)
        if data:
            self.display_current_weather(data)
        
        input(f"\n{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
    
    def _handle_forecast(self):
        """Handle forecast request"""
        city = input(f"\n{Fore.GREEN}Enter city name: {Style.RESET_ALL}").strip()
        if not city:
            print(f"{Fore.RED}❌ City name cannot be empty!{Style.RESET_ALL}")
            input(f"{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
            return
        
        data = self.get_forecast(city)
        if data:
            self.display_forecast(data)
        
        input(f"\n{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
    
    def _handle_complete_report(self):
        """Handle complete report request"""
        city = input(f"\n{Fore.GREEN}Enter city name: {Style.RESET_ALL}").strip()
        if not city:
            print(f"{Fore.RED}❌ City name cannot be empty!{Style.RESET_ALL}")
            input(f"{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
            return
        
        print(f"\n{Fore.CYAN}🔍 Fetching complete weather report for {city}...{Style.RESET_ALL}")
        
        # Get current weather
        current_data = self.get_current_weather(city)
        if current_data:
            self.display_current_weather(current_data)
        
        # Get forecast
        forecast_data = self.get_forecast(city)
        if forecast_data:
            self.display_forecast(forecast_data)
            
            # Show chart
            if self.config.show_charts:
                self.show_temperature_chart(forecast_data)
        
        input(f"\n{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
    
    def _handle_chart(self):
        """Handle chart display"""
        if not self.current_city:
            city = input(f"\n{Fore.GREEN}Enter city name: {Style.RESET_ALL}").strip()
            if not city:
                print(f"{Fore.RED}❌ City name cannot be empty!{Style.RESET_ALL}")
                input(f"{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
                return
        else:
            city = self.current_city
        
        data = self.get_forecast(city)
        if data:
            self.show_temperature_chart(data)
        
        input(f"\n{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
    
    def _handle_history(self):
        """Handle history display"""
        self.show_history()
        input(f"\n{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
    
    def _handle_save(self):
        """Handle save to file"""
        if not self.history:
            print(f"\n{Fore.RED}❌ No weather data available to save{Style.RESET_ALL}")
            input(f"{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
            return
        
        # Get last searched city
        last_entry = self.history[-1]
        city = last_entry['city']
        
        # Prepare content
        content = []
        content.append(f"Weather Report for {city}")
        content.append(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        content.append("="*60)
        
        # Add current weather if available
        if 'main' in last_entry['data']:
            content.append("\nCURRENT WEATHER:")
            content.append(f"Temperature: {format_temperature(last_entry['data']['main']['temp'])}")
            content.append(f"Condition: {last_entry['data']['weather'][0]['description'].title()}")
            content.append(f"Humidity: {last_entry['data']['main']['humidity']}%")
        
        # Save to file
        filename = save_report(city, "\n".join(content))
        if filename:
            print(f"\n{Fore.GREEN}✅ Report saved to: {filename}{Style.RESET_ALL}")
        else:
            print(f"\n{Fore.RED}❌ Failed to save report{Style.RESET_ALL}")
        
        input(f"{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
    
    def _handle_help(self):
        """Display help information"""
        clear_screen()
        
        help_text = [
            f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}",
            f"{Fore.YELLOW}📖 HELP & INSTRUCTIONS{Style.RESET_ALL}",
            f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}",
            "",
            f"{Fore.GREEN}🔑 API KEY SETUP:{Style.RESET_ALL}",
            "1. Get a free API key from https://openweathermap.org/api",
            "2. Create a file called '.env' in the same directory",
            "3. Add this line: API_KEY=your_actual_api_key_here",
            "",
            f"{Fore.GREEN}🏙️  CITY INPUT FORMAT:{Style.RESET_ALL}",
            "• Simple: 'London'",
            "• With country code: 'London,UK'",
            "• With state code (US): 'New York,US,NY'",
            "• Multiple cities same name: 'Paris,FR' or 'Paris,TX,US'",
            "",
            f"{Fore.GREEN}🌡️  TEMPERATURE UNITS:{Style.RESET_ALL}",
            "• Edit .env file: TEMPERATURE_UNIT=metric (default)",
            "• Options: metric (°C), imperial (°F), standard (K)",
            "",
            f"{Fore.GREEN}💾 SAVING REPORTS:{Style.RESET_ALL}",
            "• Reports are automatically saved in 'logs/' directory",
            "• Filename: weather_City_YYYYMMDD_HHMMSS.txt",
            "",
            f"{Fore.GREEN}📈 TEMPERATURE CHART:{Style.RESET_ALL}",
            "• Shows 5-day temperature trend",
            "• Longer bars = higher temperatures",
            "• Disabled if SHOW_CHARTS=false in .env",
            "",
            f"{Fore.CYAN}{'='*60}{Style.RESET_ALL}"
        ]
        
        print("\n".join(help_text))
        input(f"\n{Fore.YELLOW}Press Enter to continue...{Style.RESET_ALL}")
```

# src/utils.py
```
"""
Utility functions for the weather application
"""

import os
import sys
import logging
from datetime import datetime
from typing import Optional, Dict
from pathlib import Path

def clear_screen():
    """Clear terminal screen"""
    os.system('cls' if os.name == 'nt' else 'clear')

def setup_logging(log_level: str = "INFO") -> logging.Logger:
    """Setup application logging"""
    log_dir = Path("logs")
    log_dir.mkdir(exist_ok=True)
    
    logging.basicConfig(
        level=getattr(logging, log_level.upper()),
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.FileHandler(log_dir / f"weather_app_{datetime.now().strftime('%Y%m%d')}.log"),
            logging.StreamHandler()
        ]
    )
    
    return logging.getLogger(__name__)

def format_temperature(temp: float, unit: str = "metric") -> str:
    """Format temperature with unit symbol"""
    if unit == "metric":
        return f"{temp:.1f}°C"
    elif unit == "imperial":
        return f"{temp:.1f}°F"
    else:
        return f"{temp:.1f}K"

def get_weather_icon(weather_id: int) -> str:
    """Return emoji icon based on weather condition code"""
    if 200 <= weather_id <= 232:  # Thunderstorm
        return "⛈️"
    elif 300 <= weather_id <= 321:  # Drizzle
        return "🌧️"
    elif 500 <= weather_id <= 531:  # Rain
        return "🌦️"
    elif 600 <= weather_id <= 622:  # Snow
        return "❄️"
    elif 701 <= weather_id <= 781:  # Atmosphere
        return "🌫️"
    elif weather_id == 800:  # Clear
        return "☀️"
    elif 801 <= weather_id <= 804:  # Clouds
        return "☁️"
    else:
        return "🌈"

def create_ascii_chart(forecast_data: Dict):
    """Create ASCII temperature chart from forecast data"""
    try:
        # Extract temperatures for next 5 days
        temps = []
        dates = []
        
        # Get one reading per day (around midday)
        for item in forecast_data.get('list', []):
            hour = int(item['dt_txt'].split()[1].split(':')[0])
            if 11 <= hour <= 13:
                date = item['dt_txt'].split()[0]
                if date not in dates:
                    dates.append(date)
                    temps.append(item['main']['temp'])
        
        if len(temps) < 2:
            return
        
        print(f"\n📈 TEMPERATURE CHART (Next {len(temps)} days)")
        print("┌" + "─" * 50 + "┐")
        
        max_temp = max(temps)
        min_temp = min(temps)
        chart_width = 40
        
        for i, (date, temp) in enumerate(zip(dates, temps)):
            date_obj = datetime.strptime(date, '%Y-%m-%d')
            day_name = date_obj.strftime('%a')
            
            # Calculate bar length
            if max_temp > min_temp:
                bar_length = int(((temp - min_temp) / (max_temp - min_temp)) * chart_width)
            else:
                bar_length = chart_width // 2
            
            bar = "█" * max(1, bar_length)
            print(f"│ {date} ({day_name}): {bar:<{chart_width}} {temp:5.1f}°C │")
        
        print("└" + "─" * 50 + "┘")
        print(f"   Min: {min_temp:.1f}°C {' ' * 30} Max: {max_temp:.1f}°C")
        
    except Exception as e:
        logging.error(f"Failed to create chart: {e}")

def save_report(city: str, content: str) -> Optional[str]:
    """Save weather report to file"""
    try:
        log_dir = Path("logs")
        log_dir.mkdir(exist_ok=True)
        
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = log_dir / f"weather_{city}_{timestamp}.txt"
        
        with open(filename, 'w', encoding='utf-8') as f:
            f.write(content)
        
        return str(filename)
    except Exception as e:
        logging.error(f"Failed to save report: {e}")
        return None

def validate_city_name(city: str) -> bool:
    """Validate city name input"""
    if not city or len(city.strip()) < 2:
        return False
    
    # Basic validation - can be extended
    import re
    return bool(re.match(r'^[a-zA-Z\s,.-]+$', city))

def check_dependencies() -> bool:
    """Check if all required dependencies are installed"""
    required_packages = ['requests', 'python-dotenv']
    
    for package in required_packages:
        try:
            __import__(package.replace('-', '_'))
        except ImportError:
            return False
    
    return True
```

# src/config.py
```
"""
Configuration management for the weather application
"""

import os
from dataclasses import dataclass
from typing import Optional
from dotenv import load_dotenv

load_dotenv()

@dataclass
class Config:
    """Application configuration"""
    api_key: str
    temperature_unit: str = "metric"
    language: str = "en"
    show_emojis: bool = True
    show_charts: bool = True
    auto_save: bool = False
    log_level: str = "INFO"
    
    @classmethod
    def from_env(cls) -> 'Config':
        """Create config from environment variables"""
        return cls(
            api_key=os.getenv("API_KEY", ""),
            temperature_unit=os.getenv("TEMPERATURE_UNIT", "metric"),
            language=os.getenv("LANGUAGE", "en"),
            show_emojis=os.getenv("SHOW_EMOJIS", "true").lower() == "true",
            show_charts=os.getenv("SHOW_CHARTS", "true").lower() == "true",
            auto_save=os.getenv("AUTO_SAVE", "false").lower() == "true",
            log_level=os.getenv("LOG_LEVEL", "INFO")
        )

def load_config() -> Config:
    """Load configuration from environment variables"""
    return Config.from_env()
```

# scripts/setup.sh
```
#!/bin/bash

# Advanced Weather App Setup Script
# Run this on AWS EC2 or any Linux system

echo "=========================================="
echo "🌤️  Advanced Weather App Setup"
echo "=========================================="

# Update system
echo "Updating system packages..."
sudo apt-get update -y || sudo yum update -y

# Install Python and pip
echo "Installing Python and pip..."
if command -v apt-get &> /dev/null; then
    sudo apt-get install -y python3 python3-pip python3-venv
elif command -v yum &> /dev/null; then
    sudo yum install -y python3 python3-pip
fi

# Create virtual environment
echo "Creating virtual environment..."
python3 -m venv venv || echo "Virtual environment creation failed, continuing..."

# Activate virtual environment
if [ -d "venv" ]; then
    source venv/bin/activate
fi

# Install dependencies
echo "Installing Python dependencies..."
pip3 install --upgrade pip
pip3 install -r requirements.txt

# Create necessary directories
echo "Creating directories..."
mkdir -p logs
mkdir -p data

# Create .env file if it doesn't exist
if [ ! -f ".env" ]; then
    echo "Creating .env file from example..."
    cp .env.example .env
    echo ""
    echo "⚠️  IMPORTANT: Please edit .env file with your OpenWeatherMap API key"
    echo "   Get a free API key from: https://openweathermap.org/api"
    echo ""
fi

# Make scripts executable
echo "Making scripts executable..."
chmod +x scripts/*.sh 2>/dev/null || true
chmod +x src/*.py 2>/dev/null || true

# Create systemd service file (for EC2)
if [ -f /etc/systemd/system/weather-app.service ]; then
    echo "Systemd service already exists."
else
    echo "Creating systemd service..."
    sudo tee /etc/systemd/system/weather-app.service > /dev/null << EOF
[Unit]
Description=Advanced Weather Application
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=$(pwd)
Environment="PATH=$(pwd)/venv/bin:/usr/bin"
ExecStart=$(which python3) src/main.py
Restart=on-failure
RestartSec=10
StandardOutput=append:$(pwd)/logs/weather-app.log
StandardError=append:$(pwd)/logs/weather-app-error.log

[Install]
WantedBy=multi-user.target
EOF

    echo "Systemd service created at /etc/systemd/system/weather-app.service"
    echo "To enable: sudo systemctl enable weather-app"
    echo "To start: sudo systemctl start weather-app"
fi

echo ""
echo "=========================================="
echo "✅ Setup complete!"
echo "=========================================="
echo ""
echo "To run the application:"
echo "   python3 src/main.py"
echo ""
echo "To run with Docker:"
echo "   docker build -t weather-app ."
echo "   docker run -it --env-file .env weather-app"
echo ""
echo "For AWS EC2 deployment, see docs/EC2_DEPLOYMENT.md"
echo ""
```

# tests/test_weather_app.py
```
"""
Unit tests for the weather application
"""

import pytest
import sys
from pathlib import Path

# Add src directory to Python path
sys.path.insert(0, str(Path(__file__).parent.parent / 'src'))

from weather_app import WeatherApp
from utils import format_temperature, get_weather_icon, validate_city_name
from config import Config

class TestUtils:
    """Test utility functions"""
    
    def test_format_temperature(self):
        """Test temperature formatting"""
        assert format_temperature(25.5) == "25.5°C"
        assert format_temperature(25.5, "metric") == "25.5°C"
        assert format_temperature(77.9, "imperial") == "77.9°F"
        assert format_temperature(298.65, "standard") == "298.6K"
    
    def test_get_weather_icon(self):
        """Test weather icon selection"""
        assert get_weather_icon(200) == "⛈️"   # Thunderstorm
        assert get_weather_icon(300) == "🌧️"   # Drizzle
        assert get_weather_icon(500) == "🌦️"   # Rain
        assert get_weather_icon(600) == "❄️"   # Snow
        assert get_weather_icon(800) == "☀️"   # Clear
        assert get_weather_icon(801) == "☁️"   # Clouds
        assert get_weather_icon(999) == "🌈"   # Default
    
    def test_validate_city_name(self):
        """Test city name validation"""
        assert validate_city_name("London") == True
        assert validate_city_name("New York") == True
        assert validate_city_name("Paris,FR") == True
        assert validate_city_name("") == False
        assert validate_city_name("A") == False
        assert validate_city_name("123") == False

class TestConfig:
    """Test configuration management"""
    
    def test_config_creation(self):
        """Test config creation"""
        config = Config(
            api_key="test_key",
            temperature_unit="metric",
            language="en",
            show_emojis=True,
            show_charts=True,
            auto_save=False,
            log_level="INFO"
        )
        
        assert config.api_key == "test_key"
        assert config.temperature_unit == "metric"
        assert config.language == "en"
        assert config.show_emojis == True
        assert config.show_charts == True
        assert config.auto_save == False
        assert config.log_level == "INFO"

# Mock classes for testing
class MockResponse:
    """Mock API response"""
    def __init__(self, status_code=200, json_data=None):
        self.status_code = status_code
        self._json_data = json_data or {}
    
    def json(self):
        return self._json_data

if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

# EC2_DEPLOYMENT
# AWS EC2 Deployment Guide

This guide walks you through deploying the Advanced Weather App on AWS EC2.

## Prerequisites

1. **AWS Account** with EC2 access
2. **PuTTY** or SSH client
3. **OpenWeatherMap API Key**

## Step 1: Launch EC2 Instance

1. Go to AWS Console → EC2 → Launch Instance
2. Choose Amazon Linux 2023 AMI
3. Select t2.micro (Free Tier eligible)
4. Configure security group:
   - Allow SSH (port 22) from your IP
   - Allow HTTP (port 80) if adding web interface
5. Launch instance and download key pair (.pem file)

## Step 2: Connect via PuTTY

1. Convert .pem to .ppk using PuTTYgen
2. Open PuTTY
3. Enter your EC2 public IP
4. Connection → SSH → Auth → Browse and select .ppk file
5. Click Open to connect

## Step 3: Initial Setup on EC2

```bash
# Update system
sudo yum update -y

# Install Git
sudo yum install git -y

# Clone the repository
git clone https://github.com/yourusername/advanced-weather-app.git
cd advanced-weather-app

# Run setup script
chmod +x scripts/setup.sh
./scripts/setup.sh
```

# Configure API Key
```
# Edit .env file
nano .env

# Add your OpenWeatherMap API key
API_KEY=your_actual_api_key_here
```

# Run the Application
# Option A: Direct Run
```
python3 src/main.py
```

# Option B: Systemd Service (Recommended)
```
# Enable and start the service
sudo systemctl enable weather-app
sudo systemctl start weather-app

# Check status
sudo systemctl status weather-app

# View logs
tail -f logs/weather-app.log
```

# Option C: Docker
```
# Build Docker image
docker build -t weather-app .

# Run container
docker run -d --name weather-app --restart always \
  --env-file .env \
  -v $(pwd)/logs:/app/logs \
  weather-app
```

# Security Hardening
# Update Regularly
```
# Add to crontab
0 2 * * * sudo yum update -y
```

# Configure Firewall
```
sudo yum install firewalld -y
sudo systemctl start firewalld
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

# Troubleshooting
1. Can't connect via SSH
Check security group rules

Verify key pair permissions: chmod 400 your-key.pem

Check instance state in AWS Console

2. API key not working
Verify .env file format

Check API key activation (free tier takes time)

Test with: curl "api.openweathermap.org/data/2.5/weather?q=London&appid=YOUR_KEY"

3. Application won't start
Check Python version: python3 --version

Verify dependencies: pip3 list

Check logs: tail -f logs/weather-app.log

4. Memory issues on t2.micro
Monitor memory: free -h

Add swap space:
```
sudo dd if=/dev/zero of=/swapfile bs=128M count=16
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```


# Monitoring and Maintainence
# 1. Monitor application
```
# View logs
tail -f logs/weather-app.log

# Check system resources
htop

# Check disk space
df -h

# Check running processes
ps aux | grep python
```

# 2. Update Application
```
cd advanced-weather-app
git pull origin main
pip3 install -r requirements.txt --upgrade
sudo systemctl restart weather-app
```
