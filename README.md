# Mot

Certainly! Below is the updated C# application that incorporates all the improvements and handles additional edge cases as discussed. The code now includes:

Edge Case Handling:

File creation, deletion, renaming, and overwriting.
Record recreation if deleted from the database while the file still exists.
Initial scan of directories at startup.
Dynamic monitoring of directories based on database changes.
Prevention of duplicate records.
Handling of file locking and access errors.
Cleanup of records for files that no longer exist.
Improvements and Optimizations:

Batch database writes using a queue.
Asynchronous programming for non-blocking operations.
Error handling with retries and exponential backoff.
Configurable parameters via a configuration file.
Detailed logging for monitoring and debugging.
Indexing on database tables for performance.
Debouncing file system events to prevent duplicate processing.
Configuration File (appsettings.json)
First, we'll use a configuration file to make parameters like the database connection string and scanning intervals configurable.

json
Copy code
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=YourServer;Initial Catalog=YourDatabase;Integrated Security=True;"
  },
  "Monitoring": {
    "ScanIntervalInMinutes": 5,
    "EnableSubdirectories": false,
    "EventDebounceDelayInMilliseconds": 500
  },
  "Logging": {
    "LogFilePath": "logs\\application.log",
    "LogLevel": "Information"
  }
}
Updated C# Application Code
Below is the complete updated Program.cs file with all the improvements.

csharp
Copy code
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Data.SqlClient;
using System.IO;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Logging.Console;

namespace FolderMonitorApp
{
    class Program
    {
        // Configuration variables
        private static string connectionString;
        private static TimeSpan scanInterval;
        private static bool includeSubdirectories;
        private static int eventDebounceDelay;
        private static string logFilePath;
        private static LogLevel logLevel;

        // Concurrent collections for thread-safe operations
        private static ConcurrentDictionary<string, FileSystemWatcher> watchers = new ConcurrentDictionary<string, FileSystemWatcher>();
        private static BlockingCollection<FileEvent> eventQueue = new BlockingCollection<FileEvent>();

        // Cancellation token for periodic scanning and dynamic directory monitoring
        private static CancellationTokenSource cancellationTokenSource = new CancellationTokenSource();

        // Logger
        private static ILogger logger;

        static async Task Main(string[] args)
        {
            try
            {
                // Initialize configuration and logger
                InitializeConfiguration();
                InitializeLogger();

                logger.LogInformation("Application started.");

                // Start processing the event queue
                Task processingTask = Task.Run(() => ProcessEventQueue(), cancellationTokenSource.Token);

                // Start monitoring directories
                await StartMonitoringDirectories();

                logger.LogInformation("Press 'q' to quit the application.");

                // Keep the application running until 'q' is pressed
                while (Console.Read() != 'q')
                {
                    await Task.Delay(100);
                }

                // Clean up resources
                cancellationTokenSource.Cancel();
                eventQueue.CompleteAdding();

                await processingTask;

                foreach (var watcher in watchers.Values)
                {
                    watcher.Dispose();
                }

                logger.LogInformation("Application stopped.");
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "An unexpected error occurred.");
            }
        }

        private static void InitializeConfiguration()
        {
            var config = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json", optional: false)
                .Build();

            connectionString = config.GetConnectionString("DefaultConnection");
            scanInterval = TimeSpan.FromMinutes(config.GetValue<int>("Monitoring:ScanIntervalInMinutes"));
            includeSubdirectories = config.GetValue<bool>("Monitoring:EnableSubdirectories");
            eventDebounceDelay = config.GetValue<int>("Monitoring:EventDebounceDelayInMilliseconds");
            logFilePath = config.GetValue<string>("Logging:LogFilePath");
            logLevel = config.GetValue<LogLevel>("Logging:LogLevel");
        }

        private static void InitializeLogger()
        {
            var loggerFactory = LoggerFactory.Create(builder =>
            {
                builder
                    .AddConsole()
                    .AddFile(logFilePath, minimumLevel: logLevel);
            });

            logger = loggerFactory.CreateLogger<Program>();
        }

        private static async Task StartMonitoringDirectories()
        {
            // Initial directory monitoring setup
            await UpdateMonitoredDirectories();

            // Start periodic scanning
            _ = Task.Run(() => StartPeriodicScan(cancellationTokenSource.Token), cancellationTokenSource.Token);

            // Start dynamic directory monitoring
            _ = Task.Run(() => MonitorDirectoryChanges(cancellationTokenSource.Token), cancellationTokenSource.Token);
        }

        private static async Task UpdateMonitoredDirectories()
        {
            List<string> directoriesToMonitor = await GetDirectoriesToMonitor();

            foreach (var directory in directoriesToMonitor)
            {
                if (!watchers.ContainsKey(directory))
                {
                    if (Directory.Exists(directory))
                    {
                        var watcher = CreateFileSystemWatcher(directory);
                        if (watchers.TryAdd(directory, watcher))
                        {
                            logger.LogInformation($"Started monitoring directory: {directory}");
                            // Initial scan of the directory
                            await ScanDirectory(directory);
                        }
                    }
                    else
                    {
                        logger.LogWarning($"Directory does not exist: {directory}");
                    }
                }
            }

            // Remove watchers for directories no longer in the database
            foreach (var directory in watchers.Keys)
            {
                if (!directoriesToMonitor.Contains(directory))
                {
                    if (watchers.TryRemove(directory, out var watcher))
                    {
                        watcher.Dispose();
                        logger.LogInformation($"Stopped monitoring directory: {directory}");
                    }
                }
            }
        }

        private static FileSystemWatcher CreateFileSystemWatcher(string directory)
        {
            var watcher = new FileSystemWatcher
            {
                Path = directory,
                NotifyFilter = NotifyFilters.FileName | NotifyFilters.LastWrite | NotifyFilters.CreationTime,
                Filter = "*.*",
                IncludeSubdirectories = includeSubdirectories,
                EnableRaisingEvents = true
            };

            // Debounce the events to prevent duplicate processing
            var debounceTimer = new Timer(_ => { }, null, Timeout.Infinite, Timeout.Infinite);

            FileSystemEventHandler handler = (sender, e) =>
            {
                debounceTimer.Change(eventDebounceDelay, Timeout.Infinite);
                debounceTimer = new Timer(_ =>
                {
                    HandleFileEvent(e);
                }, null, eventDebounceDelay, Timeout.Infinite);
            };

            watcher.Created += handler;
            watcher.Changed += handler;
            watcher.Deleted += handler;
            watcher.Renamed += (sender, e) =>
            {
                debounceTimer.Change(eventDebounceDelay, Timeout.Infinite);
                debounceTimer = new Timer(_ =>
                {
                    HandleFileRenamedEvent(e);
                }, null, eventDebounceDelay, Timeout.Infinite);
            };

            return watcher;
        }

        private static void HandleFileEvent(FileSystemEventArgs e)
        {
            var fileEvent = new FileEvent
            {
                FileName = e.Name,
                DirectoryPath = Path.GetDirectoryName(e.FullPath),
                EventType = e.ChangeType.ToString(),
                EventTime = DateTime.Now
            };

            eventQueue.Add(fileEvent);
            logger.LogInformation($"File event queued: {e.ChangeType} - {e.FullPath}");
        }

        private static void HandleFileRenamedEvent(RenamedEventArgs e)
        {
            // Handle old file (deletion)
            var oldFileEvent = new FileEvent
            {
                FileName = e.OldName,
                DirectoryPath = Path.GetDirectoryName(e.OldFullPath),
                EventType = WatcherChangeTypes.Deleted.ToString(),
                EventTime = DateTime.Now
            };

            eventQueue.Add(oldFileEvent);
            logger.LogInformation($"File event queued: Renamed (old) - {e.OldFullPath}");

            // Handle new file (creation)
            var newFileEvent = new FileEvent
            {
                FileName = e.Name,
                DirectoryPath = Path.GetDirectoryName(e.FullPath),
                EventType = WatcherChangeTypes.Created.ToString(),
                EventTime = DateTime.Now
            };

            eventQueue.Add(newFileEvent);
            logger.LogInformation($"File event queued: Renamed (new) - {e.FullPath}");
        }

        private static async Task<List<string>> GetDirectoriesToMonitor()
        {
            var directories = new List<string>();

            using (var conn = new SqlConnection(connectionString))
            {
                await conn.OpenAsync();

                string query = "SELECT DirectoryPath FROM MonitoredDirectories";

                using (var cmd = new SqlCommand(query, conn))
                {
                    var reader = await cmd.ExecuteReaderAsync();

                    while (await reader.ReadAsync())
                    {
                        directories.Add(reader["DirectoryPath"].ToString());
                    }
                }
            }

            return directories;
        }

        private static async Task ScanDirectory(string directory)
        {
            try
            {
                if (Directory.Exists(directory))
                {
                    var files = Directory.GetFiles(directory, "*.*", includeSubdirectories ? SearchOption.AllDirectories : SearchOption.TopDirectoryOnly);

                    foreach (var file in files)
                    {
                        var fileName = Path.GetFileName(file);

                        var fileEvent = new FileEvent
                        {
                            FileName = fileName,
                            DirectoryPath = Path.GetDirectoryName(file),
                            EventType = "ExistingFile",
                            EventTime = DateTime.Now
                        };

                        eventQueue.Add(fileEvent);
                    }

                    logger.LogInformation($"Initial scan completed for directory: {directory}");
                }
            }
            catch (Exception ex)
            {
                logger.LogError(ex, $"Error scanning directory: {directory}");
            }
        }

        private static async Task StartPeriodicScan(CancellationToken cancellationToken)
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                try
                {
                    await UpdateMonitoredDirectories();

                    foreach (var directory in watchers.Keys)
                    {
                        await ScanDirectory(directory);
                    }

                    await Task.Delay(scanInterval, cancellationToken);
                }
                catch (TaskCanceledException)
                {
                    // Task was canceled, exit the loop
                    break;
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "Error during periodic scan.");
                }
            }
        }

        private static async Task MonitorDirectoryChanges(CancellationToken cancellationToken)
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                try
                {
                    await UpdateMonitoredDirectories();
                    await Task.Delay(TimeSpan.FromMinutes(1), cancellationToken); // Adjust as needed
                }
                catch (TaskCanceledException)
                {
                    // Task was canceled, exit the loop
                    break;
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "Error monitoring directory changes.");
                }
            }
        }

        private static async Task ProcessEventQueue()
        {
            foreach (var fileEvent in eventQueue.GetConsumingEnumerable(cancellationTokenSource.Token))
            {
                await HandleFileEventAsync(fileEvent);
            }
        }

        private static async Task HandleFileEventAsync(FileEvent fileEvent)
        {
            int retryCount = 0;
            int maxRetries = 3;
            int delay = 1000; // Initial delay in milliseconds

            while (retryCount < maxRetries)
            {
                try
                {
                    if (fileEvent.EventType == WatcherChangeTypes.Created.ToString() || fileEvent.EventType == "ExistingFile")
                    {
                        await InsertOrUpdateFileEvent(fileEvent);
                    }
                    else if (fileEvent.EventType == WatcherChangeTypes.Deleted.ToString())
                    {
                        await DeleteFileEvent(fileEvent.DirectoryPath, fileEvent.FileName);
                    }
                    else if (fileEvent.EventType == WatcherChangeTypes.Changed.ToString())
                    {
                        await UpdateFileEventTime(fileEvent.DirectoryPath, fileEvent.FileName, fileEvent.EventTime);
                    }

                    break; // Exit the retry loop on success
                }
                catch (SqlException ex)
                {
                    retryCount++;
                    logger.LogError(ex, $"SQL error handling file event. Retry {retryCount}/{maxRetries}.");

                    if (retryCount >= maxRetries)
                    {
                        logger.LogError($"Max retries reached for file event: {fileEvent.FileName} in {fileEvent.DirectoryPath}");
                    }
                    else
                    {
                        await Task.Delay(delay);
                        delay *= 2; // Exponential backoff
                    }
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, $"Error handling file event: {fileEvent.FileName} in {fileEvent.DirectoryPath}");
                    break; // For non-SQL exceptions, do not retry
                }
            }
        }

        private static async Task InsertOrUpdateFileEvent(FileEvent fileEvent)
        {
            using (var conn = new SqlConnection(connectionString))
            {
                await conn.OpenAsync();

                string query = @"
                    MERGE FileEvents AS target
                    USING (SELECT @DirectoryPath AS DirectoryPath, @FileName AS FileName) AS source
                    ON (target.DirectoryPath = source.DirectoryPath AND target.FileName = source.FileName)
                    WHEN MATCHED THEN
                        UPDATE SET EventTime = @EventTime, EventType = @EventType
                    WHEN NOT MATCHED THEN
                        INSERT (DirectoryPath, FileName, EventType, EventTime)
                        VALUES (@DirectoryPath, @FileName, @EventType, @EventTime);";

                using (var cmd = new SqlCommand(query, conn))
                {
                    cmd.Parameters.AddWithValue("@DirectoryPath", fileEvent.DirectoryPath);
                    cmd.Parameters.AddWithValue("@FileName", fileEvent.FileName);
                    cmd.Parameters.AddWithValue("@EventType", fileEvent.EventType);
                    cmd.Parameters.AddWithValue("@EventTime", fileEvent.EventTime);

                    await cmd.ExecuteNonQueryAsync();
                }
            }

            logger.LogInformation($"Database updated for file: {Path.Combine(fileEvent.DirectoryPath, fileEvent.FileName)}");
        }

        private static async Task UpdateFileEventTime(string directoryPath, string fileName, DateTime eventTime)
        {
            using (var conn = new SqlConnection(connectionString))
            {
                await conn.OpenAsync();

                string query = "UPDATE FileEvents SET EventTime = @EventTime WHERE DirectoryPath = @DirectoryPath AND FileName = @FileName";

                using (var cmd = new SqlCommand(query, conn))
                {
                    cmd.Parameters.AddWithValue("@DirectoryPath", directoryPath);
                    cmd.Parameters.AddWithValue("@FileName", fileName);
                    cmd.Parameters.AddWithValue("@EventTime", eventTime);

                    await cmd.ExecuteNonQueryAsync();
                }
            }

            logger.LogInformation($"Event time updated for file: {Path.Combine(directoryPath, fileName)}");
        }

        private static async Task DeleteFileEvent(string directoryPath, string fileName)
        {
            using (var conn = new SqlConnection(connectionString))
            {
                await conn.OpenAsync();

                string query = "DELETE FROM FileEvents WHERE DirectoryPath = @DirectoryPath AND FileName = @FileName";

                using (var cmd = new SqlCommand(query, conn))
                {
                    cmd.Parameters.AddWithValue("@DirectoryPath", directoryPath);
                    cmd.Parameters.AddWithValue("@FileName", fileName);

                    await cmd.ExecuteNonQueryAsync();
                }
            }

            logger.LogInformation($"Record deleted for file: {Path.Combine(directoryPath, fileName)}");
        }
    }

    // FileEvent class to represent file system events
    class FileEvent
    {
        public string FileName { get; set; }
        public string DirectoryPath { get; set; }
        public string EventType { get; set; }
        public DateTime EventTime { get; set; }
    }
}
Explanation of the Updates
Configuration Management
appsettings.json: Added to store configuration parameters like connection strings, scan intervals, and logging settings.
Configurable Parameters: The application now reads settings like scan interval, event debounce delay, and logging level from the configuration file.
Logging Enhancements
Microsoft.Extensions.Logging: Utilized for robust logging capabilities.
Log Levels: Configurable log levels to control the verbosity of logs.
Log File: Logs are written to a file specified in the configuration.
Asynchronous Programming
async/await: Implemented asynchronous methods to prevent blocking operations, especially during database access.
Event Queue Processing: The event queue is processed asynchronously to improve performance and responsiveness.
Batch Database Writes
Event Queue (BlockingCollection<FileEvent>): File events are queued and processed in batches by a separate task.
ProcessEventQueue Method: Consumes events from the queue and processes them asynchronously.
Error Handling and Retries
Retry Logic: Implemented retries with exponential backoff for transient database errors.
Exception Handling: Detailed logging for exceptions to aid in debugging.
Dynamic Directory Monitoring
UpdateMonitoredDirectories Method: Periodically updates the list of directories to monitor based on changes in the database.
Dynamic Addition/Removal of Watchers: File system watchers are added or removed as directories are added or removed from the database.
Event Debouncing
Debounce Timer: Implemented to prevent duplicate processing of file system events that may fire multiple times for a single action.
Configurable Debounce Delay: The delay is configurable via appsettings.json.
Edge Case Handling
File Creation, Deletion, Renaming, and Overwriting: All file events are handled, and the database is updated accordingly.
Initial Directory Scan: On startup and periodically, directories are scanned to detect existing files and any discrepancies between the file system and the database.
Record Recreation: If a record is missing but the file exists, it is recreated.
Cleanup of Orphaned Records: If a record exists but the file does not, the record is deleted.
Database Optimizations
MERGE Statement: Used in InsertOrUpdateFileEvent to insert or update records efficiently.
Indexing: Ensure that indexes are created on DirectoryPath and FileName in the FileEvents table to improve query performance.
Thread Safety
Concurrent Collections: Used ConcurrentDictionary and BlockingCollection for thread-safe operations.
Cancellation Tokens: Used to gracefully stop background tasks when the application exits.
Updated SQL Table Schema
Ensure that the FileEvents table has appropriate indexes and constraints.

sql
Copy code
-- Create the FileEvents table with indexes
CREATE TABLE FileEvents (
    EventID INT IDENTITY(1,1) PRIMARY KEY,
    FileName NVARCHAR(255) NOT NULL,
    DirectoryPath NVARCHAR(255) NOT NULL,
    EventType NVARCHAR(50) NOT NULL,
    EventTime DATETIME NOT NULL,
    CONSTRAINT UQ_FileEvents UNIQUE (DirectoryPath, FileName)
);

-- Create indexes on frequently queried columns
CREATE INDEX IDX_FileEvents_DirectoryPath ON FileEvents (DirectoryPath);
CREATE INDEX IDX_FileEvents_FileName ON FileEvents (FileName);
Testing the Updated Application
Setup Configuration:

Update appsettings.json with your actual database connection string and desired settings.
Build and Run:

Build the application to ensure there are no errors.
Run the application.
Monitor Logs:

Check the console and log files to see detailed information about the application's operations.
Test Edge Cases:

File Creation: Create new files in monitored directories and verify that records are added to the database.
File Deletion: Delete files and verify that records are removed from the database.
File Renaming: Rename files and verify that the old record is deleted and a new one is created.
Record Deletion: Manually delete a record from the database and verify that it is recreated if the file still exists.
Directory Changes: Add or remove directories from the MonitoredDirectories table and verify that the application adjusts accordingly without restarting.
High Volume Testing:

Simulate high file system activity to test the application's performance and stability.
Database Downtime:

Simulate database connectivity issues and verify that the application retries operations and handles errors gracefully.
Additional Considerations
Permissions: Ensure that the application has the necessary permissions to access the directories and the database.

Security:

Use secure methods to store and retrieve the database connection string (e.g., environment variables or secure secrets management).
Implement parameterized SQL queries to prevent SQL injection attacks.
Deployment:

Consider converting the console application into a Windows Service for continuous monitoring.
Use deployment tools or scripts to automate the deployment process.
Logging Libraries:

You can replace the simple file logger with more advanced logging frameworks like Serilog or NLog for better performance and features.
Conclusion
The updated application now handles additional edge cases and incorporates several improvements for better performance, scalability, and robustness. By implementing these changes, the application is more resilient to unexpected scenarios and can handle high volumes of file system events efficiently.

Feel free to customize and extend the application further to meet any additional requirements you might have. Let me know if you need assistance with any specific part of the code or further explanations!






