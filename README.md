"""
Water Quality Analysis System
Author: Ruth Cheruto
Date: 2024
Description: A Python program to analyze water quality data from CSV files,
             check safety thresholds, and generate comprehensive reports.
"""

import csv
from statistics import mean
from typing import List, Dict, Tuple, Optional
import sys
import os

# Safe parameter thresholds based on environmental standards
SAFE_THRESHOLDS = {
    "ph": (6.5, 8.5),           # pH range for safe drinking water
    "turbidity": (0.0, 1.0),    # NTU - Nephelometric Turbidity Units
    "temperature": (0.0, 35.0)  # Celsius - reasonable water temp range
}


class WaterQualityAnalyzer:
    """
    A class to analyze water quality data and generate safety reports.
    """
    
    def __init__(self, file_path: str):
        """
        Initialize the analyzer with a data file path.
        
        Args:
            file_path (str): Path to the CSV file containing water quality data
        """
        self.file_path = file_path
        self.data = []
        self.alerts = []
        self.stats = {}
    
    def read_data(self) -> List[Dict]:
        """
        Read and parse CSV data from the specified file.
        
        Returns:
            List[Dict]: List of dictionaries containing water quality readings
            
        Raises:
            FileNotFoundError: If the specified file doesn't exist
            ValueError: If the file is empty or has invalid structure
        """
        print(f"📖 Reading data from: {self.file_path}")
        
        if not os.path.exists(self.file_path):
            raise FileNotFoundError(f"Data file not found: {self.file_path}")
        
        data = []
        skipped_rows = 0
        
        try:
            with open(self.file_path, 'r', encoding='utf-8') as file:
                # Check if file is empty
                if os.path.getsize(self.file_path) == 0:
                    raise ValueError("The data file is empty")
                
                reader = csv.DictReader(file)
                
                # Validate required columns
                required_columns = ['timestamp', 'location', 'ph', 'turbidity', 'temperature']
                if not all(col in reader.fieldnames for col in required_columns):
                    missing = [col for col in required_columns if col not in reader.fieldnames]
                    raise ValueError(f"Missing required columns: {missing}")
                
                for row_num, row in enumerate(reader, 2):  # Start from 2 (header is row 1)
                    try:
                        # Convert numeric fields and validate
                        processed_row = self._process_row(row, row_num)
                        if processed_row:
                            data.append(processed_row)
                        else:
                            skipped_rows += 1
                            
                    except (ValueError, KeyError) as e:
                        print(f"⚠️ Skipping row {row_num}: {e}")
                        skipped_rows += 1
                        continue
            
            if not data:
                raise ValueError("No valid data rows found in the file")
                
            print(f"✅ Successfully loaded {len(data)} records")
            if skipped_rows > 0:
                print(f"⚠️ Skipped {skipped_rows} invalid records")
                
            self.data = data
            return data
            
        except csv.Error as e:
            raise ValueError(f"CSV parsing error: {e}")
    
    def _process_row(self, row: Dict, row_num: int) -> Optional[Dict]:
        """
        Process and validate a single row of data.
        
        Args:
            row (Dict): Raw row data from CSV
            row_num (int): Row number for error reporting
            
        Returns:
            Optional[Dict]: Processed row or None if invalid
        """
        processed_row = row.copy()
        
        # Convert numeric fields
        for field in ['ph', 'turbidity', 'temperature']:
            try:
                value = row[field].strip() if row[field] else None
                if not value:
                    raise ValueError(f"Missing value for {field}")
                processed_row[field] = float(value)
            except ValueError:
                raise ValueError(f"Invalid numeric value for {field}: '{row[field]}'")
        
        # Basic validation
        if not row['timestamp'] or not row['timestamp'].strip():
            raise ValueError("Missing timestamp")
        
        if not row['location'] or not row['location'].strip():
            raise ValueError("Missing location")
        
        return processed_row
    
    def compute_statistics(self) -> Dict[str, Dict]:
        """
        Compute comprehensive statistics for all water quality parameters.
        
        Returns:
            Dict: Statistics for each parameter including min, max, average
        """
        print("📊 Computing statistics...")
        
        stats = {}
        parameters = ['ph', 'turbidity', 'temperature']
        
        for param in parameters:
            values = [record[param] for record in self.data]
            
            stats[param] = {
                'min': min(values),
                'max': max(values),
                'average': mean(values),
                'count': len(values),
                'safe_min': SAFE_THRESHOLDS[param][0],
                'safe_max': SAFE_THRESHOLDS[param][1]
            }
        
        self.stats = stats
        return stats
    
    def check_safety_thresholds(self) -> List[Dict]:
        """
        Check all readings against safety thresholds and generate alerts.
        
        Returns:
            List[Dict]: List of alert dictionaries for unsafe readings
        """
        print("🔍 Checking safety thresholds...")
        
        alerts = []
        
        for record in self.data:
            for param, (low, high) in SAFE_THRESHOLDS.items():
                value = record[param]
                
                if value < low or value > high:
                    alert = {
                        'timestamp': record['timestamp'],
                        'location': record['location'],
                        'parameter': param,
                        'value': value,
                        'threshold_low': low,
                        'threshold_high': high,
                        'issue': f"{param.upper()} too low" if value < low else f"{param.upper()} too high"
                    }
                    alerts.append(alert)
        
        self.alerts = alerts
        return alerts
    
    def generate_report(self) -> str:
        """
        Generate a comprehensive water quality report.
        
        Returns:
            str: Formatted report string
        """
        if not self.stats:
            self.compute_statistics()
        
        if not self.alerts:
            self.check_safety_thresholds()
        
        report_lines = []
        
        # Header
        report_lines.append("=" * 60)
        report_lines.append("           WATER QUALITY ANALYSIS REPORT")
        report_lines.append("=" * 60)
        report_lines.append(f"Data Source: {self.file_path}")
        report_lines.append(f"Total Records: {len(self.data)}")
        report_lines.append(f"Analysis Date: {self._get_current_timestamp()}")
        report_lines.append("")
        
        # Statistics Section
        report_lines.append("📈 WATER QUALITY STATISTICS")
        report_lines.append("-" * 40)
        
        for param, values in self.stats.items():
            report_lines.append(
                f"{param.upper():<12} → "
                f"Min: {values['min']:6.2f}, "
                f"Max: {values['max']:6.2f}, "
                f"Avg: {values['average']:6.2f} | "
                f"Safe Range: [{values['safe_min']}-{values['safe_max']}]"
            )
        
        report_lines.append("")
        
        # Alerts Section
        report_lines.append("🚨 SAFETY ALERTS")
        report_lines.append("-" * 40)
        
        if self.alerts:
            report_lines.append(f"Found {len(self.alerts)} unsafe reading(s):")
            report_lines.append("")
            
            # Group alerts by location for better readability
            alerts_by_location = {}
            for alert in self.alerts:
                location = alert['location']
                if location not in alerts_by_location:
                    alerts_by_location[location] = []
                alerts_by_location[location].append(alert)
            
            for location, loc_alerts in alerts_by_location.items():
                report_lines.append(f"📍 {location}:")
                for alert in loc_alerts:
                    report_lines.append(
                        f"   ⚠️ {alert['timestamp']} | "
                        f"{alert['parameter'].upper()} = {alert['value']:.2f} | "
                        f"{alert['issue']}"
                    )
                report_lines.append("")
        else:
            report_lines.append("✅ All readings are within safe limits!")
            report_lines.append("")
        
        # Summary Section
        report_lines.append("📋 SUMMARY")
        report_lines.append("-" * 40)
        safe_percentage = ((len(self.data) - len(self.alerts)) / len(self.data)) * 100
        report_lines.append(f"Safe Readings: {len(self.data) - len(self.alerts)}/{len(self.data)} ({safe_percentage:.1f}%)")
        report_lines.append(f"Unsafe Readings: {len(self.alerts)}/{len(self.data)} ({100 - safe_percentage:.1f}%)")
        
        if self.alerts:
            # Most common issue
            issues = [alert['parameter'] for alert in self.alerts]
            most_common_issue = max(set(issues), key=issues.count)
            report_lines.append(f"Most Common Issue: {most_common_issue.upper()}")
        
        report_lines.append("=" * 60)
        
        return "\n".join(report_lines)
    
    def _get_current_timestamp(self) -> str:
        """Get current timestamp for reporting."""
        from datetime import datetime
        return datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    def export_to_csv(self, output_file: str = "water_quality_report.csv") -> None:
        """
        Export analysis results to a CSV file.
        
        Args:
            output_file (str): Path for the output CSV file
        """
        try:
            with open(output_file, 'w', newline='', encoding='utf-8') as file:
                writer = csv.writer(file)
                
                # Write header
                writer.writerow(["Water Quality Analysis Report"])
                writer.writerow([f"Generated on: {self._get_current_timestamp()}"])
                writer.writerow([f"Data source: {self.file_path}"])
                writer.writerow([])
                
                # Write statistics
                writer.writerow(["STATISTICS"])
                writer.writerow(["Parameter", "Min", "Max", "Average", "Safe Min", "Safe Max"])
                for param, stats in self.stats.items():
                    writer.writerow([
                        param.upper(),
                        f"{stats['min']:.2f}",
                        f"{stats['max']:.2f}",
                        f"{stats['average']:.2f}",
                        f"{stats['safe_min']:.2f}",
                        f"{stats['safe_max']:.2f}"
                    ])
                
                writer.writerow([])
                
                # Write alerts
                writer.writerow(["SAFETY ALERTS"])
                if self.alerts:
                    writer.writerow(["Timestamp", "Location", "Parameter", "Value", "Issue"])
                    for alert in self.alerts:
                        writer.writerow([
                            alert['timestamp'],
                            alert['location'],
                            alert['parameter'].upper(),
                            f"{alert['value']:.2f}",
                            alert['issue']
                        ])
                else:
                    writer.writerow(["All readings are within safe limits!"])
                
                writer.writerow([])
                
                # Write summary
                safe_count = len(self.data) - len(self.alerts)
                safe_percentage = (safe_count / len(self.data)) * 100
                writer.writerow(["SUMMARY"])
                writer.writerow(["Total Records", len(self.data)])
                writer.writerow(["Safe Records", safe_count])
                writer.writerow(["Unsafe Records", len(self.alerts)])
                writer.writerow(["Safety Percentage", f"{safe_percentage:.1f}%"])
            
            print(f"💾 Report exported to: {output_file}")
            
        except Exception as e:
            print(f"❌ Error exporting to CSV: {e}")


def main():
    """
    Main function to run the water quality analysis.
    """
    print("💧 Water Quality Analysis System")
    print("=" * 40)
    
    # Get file path from command line or use default
    if len(sys.argv) > 1:
        file_path = sys.argv[1]
    else:
        file_path = "water_samples.csv"
    
    try:
        # Initialize and run analysis
        analyzer = WaterQualityAnalyzer(file_path)
        
        # Load data
        data = analyzer.read_data()
        
        # Compute statistics
        analyzer.compute_statistics()
        
        # Check safety thresholds
        analyzer.check_safety_thresholds()
        
        # Generate and display report
        report = analyzer.generate_report()
        print(report)
        
        # Export to CSV
        output_file = "water_quality_analysis_report.csv"
        analyzer.export_to_csv(output_file)
        
        print(f"\n🎉 Analysis completed successfully!")
        print(f"📁 Results saved to: {output_file}")
        
    except FileNotFoundError as e:
        print(f"❌ Error: {e}")
        print("\n💡 Please ensure:")
        print("   - The CSV file exists in the current directory")
        print("   - Or specify the file path as an argument:")
        print("     python water_quality_analyzer.py path/to/your/data.csv")
        sys.exit(1)
        
    except ValueError as e:
        print(f"❌ Data Error: {e}")
        sys.exit(1)
        
    except Exception as e:
        print(f"❌ Unexpected error: {e}")
        sys.exit(1)


# Sample data creation function for testing
def create_sample_data():
    """
    Create a sample CSV file for testing if it doesn't exist.
    """
    sample_data = [
        ["timestamp", "location", "ph", "turbidity", "temperature"],
        ["2024-01-15 08:00", "Lake A", "7.2", "0.9", "25"],
        ["2024-01-15 09:00", "Lake B", "8.7", "1.1", "23"],
        ["2024-01-15 10:00", "Lake C", "6.8", "0.5", "22"],
        ["2024-01-15 11:00", "Lake D", "7.1", "2.5", "24"],  # High turbidity
        ["2024-01-15 12:00", "Lake E", "6.4", "0.8", "20"],  # Low pH
        ["2024-01-15 13:00", "Lake F", "7.5", "0.3", "36"],  # High temperature
        ["2024-01-15 14:00", "Lake G", "5.9", "1.2", "18"],  # Low pH
        ["2024-01-15 15:00", "Lake H", "7.0", "0.7", "26"]   # Safe
    ]
    
    with open("water_samples.csv", "w", newline='', encoding='utf-8') as file:
        writer = csv.writer(file)
        writer.writerows(sample_data)
    
    print("📝 Created sample data file: water_samples.csv")


if __name__ == "__main__":
    # Create sample data if it doesn't exist
    if not os.path.exists("water_samples.csv"):
        print("📝 Sample data file not found. Creating one for testing...")
        create_sample_data()
        print("")
    
    main()
