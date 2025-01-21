# Namespace your templates
When creating a templates folder, if it's inside the app use namespacing to make it easier to find.
/pages/templates/pages.

# Use names in URLs
Pass in a name arg when defining a url and then use that in templates so you only need to update the url in one place
...
path("", home_page_view, name="home"),
...
 <a href="{% url 'home' %}">Home</a>

# Add __str__ to models to improve readibility
Adding def __str__(self): will update the admin page so we should always include this in models even if it's quick and simple.

# Use absolute URL with models to better link to detail pages
In your models file add something like this.
def get_absolute_url(self):
    return reverse("post_detail", kwargs={"pk": self.pk})
Then where you want to link in a template you can call
href="{% post.get_absolute_url %}"

# When using CustomUser

There are two ways to refer to a custom user model: AUTH_USER_MODEL and get_user_model. As general advice:

AUTH_USER_MODEL makes sense for references within a models.py file
get_user_model() is recommended everywhere else, such as views, tests, etc.




Unrelated F1 stuff

```
import fastf1
import pandas as pd

def get_season_results(year):
    """
    Fetch all race results for a given F1 season using FastF1.

    Parameters:
    year (int): The year to fetch results for

    Returns:
    pandas.DataFrame: Compiled results for all races in the season
    """
    # Enable caching for faster subsequent loads
    fastf1.Cache.enable_cache('f1_cache')

    # Get the schedule for the season
    schedule = fastf1.get_event_schedule(year)

    # Initialize list to store results
    season_results = []

    # Iterate through each race
    for index, race in schedule.iterrows():
        try:
            # Load the race session
            race_session = fastf1.get_session(year, race['RoundNumber'], 'R')
            race_session.load()

            # Get results and add race info
            results = race_session.results
            results['RaceName'] = race['EventName']
            results['Round'] = race['RoundNumber']

            # Add to our list
            season_results.append(results)

            print(f"Processed {race['EventName']}")

        except Exception as e:
            print(f"Error processing {race['EventName']}: {str(e)}")

    # Combine all results
    if season_results:
        all_results = pd.concat(season_results, ignore_index=True)

        # Reorder columns for better readability
        column_order = [
            'Round', 'RaceName', 'Position', 'Points',
            'DriverNumber', 'DriverCode', 'TeamName',
            'Status', 'Time', 'GridPosition'
        ]

        # Select only columns that exist (some might be missing in older races)
        existing_columns = [col for col in column_order if col in all_results.columns]
        all_results = all_results[existing_columns]

        return all_results

    return None

# Example usage
if __name__ == "__main__":
    results_2023 = get_season_results(2023)

    if results_2023 is not None:
        # Display total points per driver
        driver_points = results_2023.groupby('DriverCode')['Points'].sum().sort_values(ascending=False)
        print("\nDriver Championship Standings:")
        print(driver_points)

        # Save to CSV
        results_2023.to_csv(f'f1_results_2023.csv', index=False)
```

Quali results
```
import fastf1
import pandas as pd
import numpy as np
from datetime import datetime

def get_qualifying_results(year):
    """
    Fetch detailed qualifying results for all races in a season.

    Parameters:
    year (int): The F1 season year

    Returns:
    tuple: (qualifying_results DataFrame, detailed_times DataFrame)
    """
    fastf1.Cache.enable_cache('f1_cache')
    schedule = fastf1.get_event_schedule(year)

    qualifying_results = []
    detailed_times = []

    for index, race in schedule.iterrows():
        try:
            print(f"Processing {race['EventName']} qualifying...")

            # Load qualifying session
            quali = fastf1.get_session(year, race['RoundNumber'], 'Q')
            quali.load()

            # Get final qualifying results
            results = quali.results
            results['RaceName'] = race['EventName']
            results['Round'] = race['RoundNumber']
            qualifying_results.append(results)

            # Get detailed timing data for each driver
            for driver in quali.drivers:
                driver_laps = quali.laps.pick_driver(driver)
                if not driver_laps.empty:
                    # Get best lap for each qualifying session (Q1, Q2, Q3)
                    for session in ['Q1', 'Q2', 'Q3']:
                        session_laps = driver_laps[driver_laps['SessionType'] == session]
                        if not session_laps.empty:
                            best_lap = session_laps.pick_fastest()

                            lap_data = {
                                'Round': race['RoundNumber'],
                                'RaceName': race['EventName'],
                                'Driver': driver,
                                'Session': session,
                                'LapTime': best_lap['LapTime'],
                                'Sector1Time': best_lap['Sector1Time'],
                                'Sector2Time': best_lap['Sector2Time'],
                                'Sector3Time': best_lap['Sector3Time'],
                                'Compound': best_lap['Compound'],
                                'TyreLife': best_lap['TyreLife'],
                                'TrackStatus': best_lap['TrackStatus'],
                                'LapStartTime': best_lap['LapStartTime']
                            }
                            detailed_times.append(lap_data)

        except Exception as e:
            print(f"Error processing {race['EventName']}: {str(e)}")

    # Combine all results
    if qualifying_results and detailed_times:
        all_results = pd.concat(qualifying_results, ignore_index=True)
        all_times = pd.DataFrame(detailed_times)

        # Convert timedelta to seconds for easier calculations
        all_times['LapTimeSeconds'] = all_times['LapTime'].dt.total_seconds()

        # Add team information to detailed times
        team_mapping = all_results.set_index('DriverNumber')['TeamName'].to_dict()
        all_times['Team'] = all_times['Driver'].map(team_mapping)

        return all_results, all_times

    return None, None

def analyze_teammate_gaps(qualifying_data, detailed_times):
    """
    Analyze qualifying gaps between teammates across the season.
    """
    gaps = []

    # Group by team and race
    for (round_num, race_name), race_data in detailed_times.groupby(['Round', 'RaceName']):
        for team, team_data in race_data.groupby('Team'):
            # Only analyze if we have data for both drivers
            if len(team_data['Driver'].unique()) == 2:
                drivers = team_data['Driver'].unique()

                # Compare best times in each session
                for session in ['Q1', 'Q2', 'Q3']:
                    session_data = team_data[team_data['Session'] == session]

                    if len(session_data) == 2:  # Both drivers set a time
                        times = session_data.set_index('Driver')['LapTimeSeconds']
                        gap = times[drivers[0]] - times[drivers[1]]

                        gaps.append({
                            'Round': round_num,
                            'RaceName': race_name,
                            'Team': team,
                            'Session': session,
                            'Driver1': drivers[0],
                            'Driver2': drivers[1],
                            'Gap': abs(gap),
                            'FasterDriver': drivers[0] if gap < 0 else drivers[1]
                        })

    return pd.DataFrame(gaps)

# Example usage
if __name__ == "__main__":
    year = 2023
    quali_results, detailed_times = get_qualifying_results(year)

    if quali_results is not None and detailed_times is not None:
        # Save raw data
        quali_results.to_csv(f'qualifying_results_{year}.csv', index=False)
        detailed_times.to_csv(f'qualifying_detailed_times_{year}.csv', index=False)

        # Analyze teammate gaps
        gaps_analysis = analyze_teammate_gaps(quali_results, detailed_times)

        # Calculate average gaps between teammates
        average_gaps = gaps_analysis.groupby('Team')['Gap'].mean().sort_values()
        print("\nAverage qualifying gaps between teammates (seconds):")
        print(average_gaps)

        # Save gap analysis
        gaps_analysis.to_csv(f'qualifying_gaps_{year}.csv', index=False)
```
Since last checked
```
import fastf1
import pandas as pd
import json
from datetime import datetime
import os
from pathlib import Path

class F1ResultsTracker:
    def __init__(self, year, base_path='f1_data'):
        """
        Initialize the F1 results tracker

        Parameters:
        year (int): The F1 season year
        base_path (str): Directory to store data files
        """
        self.year = year
        self.base_path = Path(base_path)
        self.state_file = self.base_path / 'tracker_state.json'
        self.results_file = self.base_path / f'race_results_{year}.csv'

        # Create directory if it doesn't exist
        self.base_path.mkdir(exist_ok=True)

        # Enable FastF1 cache
        fastf1.Cache.enable_cache(str(self.base_path / 'f1_cache'))

        # Initialize or load state
        self.state = self._load_state()

    def _load_state(self):
        """Load the last checked race state from file"""
        if self.state_file.exists():
            with open(self.state_file, 'r') as f:
                return json.load(f)
        return {'last_checked_round': 0, 'year': self.year}

    def _save_state(self, last_round):
        """Save the current state to file"""
        state = {
            'last_checked_round': last_round,
            'year': self.year,
            'last_update': datetime.now().isoformat()
        }
        with open(self.state_file, 'w') as f:
            json.dump(state, f, indent=2)

    def _load_existing_results(self):
        """Load existing race results if any"""
        if self.results_file.exists():
            return pd.read_csv(self.results_file)
        return None

    def check_new_results(self):
        """
        Check for and fetch any new race results since last check

        Returns:
        tuple: (bool indicating if new data was found, DataFrame of new results)
        """
        print(f"Checking for new F1 results for {self.year}...")

        # Get current schedule
        schedule = fastf1.get_event_schedule(self.year)

        # Get completed races (where date has passed)
        current_date = datetime.now()
        completed_races = schedule[
            pd.to_datetime(schedule['Session5Date']) < current_date
        ]

        if completed_races.empty:
            print("No completed races found")
            return False, None

        latest_completed_round = completed_races['RoundNumber'].max()
        last_checked_round = self.state['last_checked_round']

        # Check if we have new races to process
        if latest_completed_round > last_checked_round:
            print(f"Found new results! Processing rounds {last_checked_round + 1} to {latest_completed_round}")

            new_results = []

            # Process each new race
            for round_num in range(last_checked_round + 1, latest_completed_round + 1):
                try:
                    race = schedule[schedule['RoundNumber'] == round_num].iloc[0]

                    # Load the race session
                    race_session = fastf1.get_session(self.year, round_num, 'R')
                    race_session.load()

                    # Get results and add race info
                    results = race_session.results
                    results['RaceName'] = race['EventName']
                    results['Round'] = race['RoundNumber']
                    results['Date'] = race['Session5Date']

                    new_results.append(results)
                    print(f"Processed {race['EventName']}")

                except Exception as e:
                    print(f"Error processing round {round_num}: {str(e)}")
                    continue

            if new_results:
                # Combine new results
                new_results_df = pd.concat(new_results, ignore_index=True)

                # Load and update existing results if any
                existing_results = self._load_existing_results()
                if existing_results is not None:
                    updated_results = pd.concat([existing_results, new_results_df], ignore_index=True)
                else:
                    updated_results = new_results_df

                # Save updated results
                updated_results.to_csv(self.results_file, index=False)

                # Update state
                self._save_state(latest_completed_round)

                return True, new_results_df

        else:
            print("No new results since last check")
            return False, None

# Example usage
if __name__ == "__main__":
    tracker = F1ResultsTracker(2024)
    new_data_found, new_results = tracker.check_new_results()

    if new_data_found:
        print("\nNew results summary:")
        for race in new_results['RaceName'].unique():
            race_results = new_results[new_results['RaceName'] == race]
            print(f"\n{race} Podium:")
            podium = race_results[race_results['Position'] <= 3].sort_values('Position')
            for _, driver in podium.iterrows():
                print(f"{driver['Position']}. {driver['DriverCode']} ({driver['TeamName']})")
```
