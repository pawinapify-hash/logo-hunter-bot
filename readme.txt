================================================================================
                        LOGO HUNTER BOT v8.2.1
                   Intelligent Logo Detection System
================================================================================

OVERVIEW
--------
Logo Hunter is an automated bot that searches videos and posts across multiple
social media platforms (YouTube, TikTok, Instagram, Facebook), then uses an
AI deep learning model (PyTorch) to detect a specific brand logo (BigC) in
the video frames and thumbnail images. Results are logged to Google Sheets
and evidence screenshots are uploaded to cloud storage.

================================================================================
HOW IT WORKS - STEP BY STEP
================================================================================

1. INITIALIZATION
   - Auto-installs required Python packages: yt-dlp, apify-client, gspread,
     opencv-python, torch, torchvision
   - Loads API keys from environment variables:
       * YOUTUBE_API_KEYS (comma-separated)
       * YOUTUBE_COOKIES (optional Netscape-format cookies.txt content for yt-dlp)
       * APIFY_TOKENS (comma-separated)
       * IMGBB_API_KEY (optional, for image hosting)
       * SHEET_ID (Google Sheets spreadsheet ID)
       * GDRIVE_FOLDER_ID (Google Drive folder for evidence storage)
   - Authenticates with Google Sheets and Google Drive using credentials.json
   - Loads the AI model from bigc_model.pth
   - GitHub Actions uses Python 3.11 and Deno for yt-dlp's JavaScript support.
     If YouTube requires sign-in, set the optional YOUTUBE_COOKIES GitHub Secret
     to the contents of a Netscape-format cookies.txt file. Treat these cookies
     as credentials; they are written to a temporary file during the run and
     removed on normal shutdown. YouTube cookies may expire or be rejected.

2. READ CONFIGURATION
   - Connects to Google Sheets (target spreadsheet)
   - Reads settings from the "Control_Panel" worksheet (column B):
     Row 1 -> Run Status (Start/Stop)
     Row 2 -> Platforms (YouTube, TikTok, Instagram, Facebook)
     Row 3 -> Search Keywords (comma-separated)
     Row 4 -> Time Filter (1=1 Day, 2=7 Days, 3=30 Days)
     Row 5 -> Run Mode (Run Once / Continuous)
     Row 6 -> Interval (minutes between scans, Colab only)
     Row 7 -> Max Results per platform
   - Sends heartbeat timestamp to Control_Panel to confirm bot is alive

3. FETCH CONTENT
   - For each platform/keyword combination, searches for recent content:

     YouTube:
       * Uses YouTube Data API v3 to search for videos matching keywords
       * Filters by publish date based on the time filter
       * Cycles through multiple API keys if quota is exceeded

     TikTok:
       * Uses Apify actor "clockworks/tiktok-scraper" to search by
         hashtag or keyword
       * Supports proxy rotation for reliability

     Instagram:
       * Uses Apify actor "apify/instagram-hashtag-scraper"

     Facebook:
       * Uses Apify actor "apify/facebook-search-scraper"

4. DEDUPLICATION
   - Compares fetched content against:
       * Previously processed URLs stored in Google Sheets (column G)
       * In-memory scanned URLs (to avoid re-scanning in same session)
       * Content older than the time filter cutoff date
   - Sorts content into two categories: Unique (new) and Duplicate (old/seen)

5. AI LOGO DETECTION
   For each unique video/post, the bot performs a two-phase scan:

   Phase A - Video Frame Analysis:
     * Downloads the video stream URL via yt-dlp
     * Uses OpenCV to extract frames at 30-frame intervals (up to 150 frames)
     * Passes each frame through the PyTorch AI model
     * Model classifies whether the BigC logo is present (confidence > 85%)

   Phase B - Thumbnail/Image Analysis (fallback):
     * If no logo found in video, downloads the thumbnail image
     * Runs the AI model on the static image

6. SAVE EVIDENCE IMAGES
   When a logo is detected, the evidence screenshot is uploaded via a
   multi-tier fallback system:
     Step 1 -> ImgBB (if API key configured)
     Step 2 -> Google Drive (if folder ID set and not full)
     Step 3 -> Catbox.moe (free anonymous upload)
     Step 4 -> Tmpfiles.org (free anonymous upload)

   The upload returns a URL which is embedded as a HYPERLINK formula in
   Google Sheets for easy one-click viewing.

7. LOG RESULTS TO GOOGLE SHEETS
   All results are batch-inserted into the "Apify" worksheet with:
     Column A -> Timestamp (Bangkok timezone, UTC+7)
     Column B -> Video/Post Title
     Column C -> Platform
     Column D -> User/Author
     Column E -> Detection Status (Yes/No)
     Column F -> Timestamp in video where logo was found
     Column G -> URL of the video/post
     Column H -> HYPERLINK formula to evidence image
     Column I -> Checkbox (FALSE by default)
     Column J -> Record timestamp

   A summary log is also written to the "Scan_Logs" worksheet with:
     - Timestamp, Duplicate List, Unique List, Platforms

8. SEND NOTIFICATIONS
   - Generates a summary message in Thai language containing:
       * Total posts fetched, new posts, logo detections
       * Links to posts where logos were found
       * Links to evidence images
   - Sends to Google Chat webhook (main channel)
   - Sends TikTok-specific results to a separate TikTok webhook

9. LOOP / EXIT
   - If Run Mode is "Run Once":
       * Sets status to Stop and exits the program
   - If running on GitHub Actions (serverless):
       * Runs one cycle and exits (external scheduler triggers next run)
   - If running on Google Colab (continuous mode):
       * Waits for the configured interval, then repeats from Step 3

================================================================================
ENVIRONMENT VARIABLES (Required)
================================================================================

  YOUTUBE_API_KEYS    Comma-separated YouTube Data API v3 keys
  APIFY_TOKENS        Comma-separated Apify API tokens
  SHEET_ID            Google Sheets spreadsheet ID
  GDRIVE_FOLDER_ID    Google Drive folder ID for storing evidence images
  IMGBB_API_KEY       (Optional) ImgBB API key for image hosting

================================================================================
REQUIRED FILES
================================================================================

  credentials.json    Google Service Account credentials for Sheets & Drive API
  bigc_model.pth      Trained PyTorch model for logo classification

================================================================================
GOOGLE SHEETS STRUCTURE
================================================================================

  Control_Panel    Bot configuration (Status, Platforms, Keywords, etc.)
  Apify            Main data output - detection results for each post
  Scan_Logs        Session logs (timestamps, duplicates, new items)

================================================================================
KEY FEATURES
================================================================================

  - Multi-platform search (YouTube, TikTok, Instagram, Facebook)
  - AI-powered logo detection using PyTorch deep learning
  - Multi-tier evidence image upload (ImgBB -> Drive -> Catbox -> Tmpfiles)
  - API key rotation for YouTube quota and Apify credit management
  - Google Sheets integration with batch insert for performance
  - Google Chat webhook notifications with summary reports
  - Duplicate detection to avoid re-scanning previously processed content
  - Time-based filtering (1 day, 7 days, or 30 days)
  - Heartbeat system to monitor bot health
  - Google Sheets API retry with automatic recovery (handles Error 503)
  - Works on both GitHub Actions (serverless) and Google Colab (continuous)
  - Hard limit on results per search to control resource usage
  - Bangkok timezone (UTC+7) for all timestamps
