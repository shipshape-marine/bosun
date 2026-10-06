# Bosun
<head>
  <meta name="description" content="Bosun is a local server network for the Solar Proa vessel, designed to collect and visualize data from various sensors and microcontrollers.">
  <meta name="keywords" content="Bosun, Solar Proa, Local Server, ESP32, Raspberry Pi, Data Visualization, Sensor Data">
  <meta name="author" content="Inverated">
  <meta name="robots" content="index, follow">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>


# Table of Contents
- [Technology Stack](#technology-stack)
  - [Frontend & Backend Communication](#frontend--backend-communication)
- [Backend Structure](#backend-structure)
  - [Folder Structure](#folder-structure)
  - [Steps to Add New Data Stream Handler](#steps-to-add-new-data-stream-handler)
- [Frontend Structure](#frontend-structure)
  - [Steps to add new tabs](#steps-to-add-new-tabs)
  - [Dev panel control](#dev-panel-control)
- [Running the App](#running-the-app)
- [Additional Notes](#additional-notes)


# Technology Stack

ExpressJs backend with direct connection to the master ESP32 Node via Serial communication.

React frontend for real time data visualization and device control.

## Frontend & Backend Communication

- Access to the web application via the local network on port 4000.

- Real time console management in dev panel via WebSocket & xterm to the React frontend running on port 3001 for communication

- Data streaming from backend to frontend via a one-way Server Side Event (SSE) connection for real time data visualization.

- API endpoint for ad-hoc commands / data retrieval from the backend to the React frontend.

- API endpoint with middleware for authentication and authorization for device control and access to dev panel.

## Offline map and bathymetry layers

The GPS Route map loads its base tiles from `/map-tiles/{z}/{x}/{y}.png` and its map bounds from `/map_config`. Optional bathymetry data is discovered from `proa_advisor/model/bathymetry/` by matching a pair of files with the same prefix:

- `{prefix}_ascii.asc` - WGS84 ESRI ASCII grid used for coordinate/depth sampling.
- `{prefix}_png_cm.png` - color-map PNG rendered over the base tiles.

To set up
1. Goto [OpenTopoMap](https://opentopomap.org) or other map provider and record the coordinates of the top left and bottom right corners of the area you want. 
2. Fill up the details into .env file and run `npm run download:map` to download the map tiles for offline usage (set other config such as download source and zoom level in .env file).
3. Goto [GEBCO](https://download.gebco.net/), click on "Select subset option" and fill up the coords (North, South : Lat, East, West : Long) to select the region.
4. Select Bathymetry Layer, ASCII and Color Map data to export. 
5. Unzip and paste the files into `proa_advisor/model/bathymetry/`.
6. Restart the backend to load the new bathymetry data.

Bathymetry is optional. If the paired files are missing or invalid, the backend logs a warning, the layer is reported as unavailable, and the map continues to render its base tiles and GPS route without the overlay. 

**Note: GEBCO data must not be used for navigation or any purpose relating to safety at sea.**

# Backend Structure

Backend structure is kept simple as this meant to mimic a control panel and data visualisation dashboard instead of a fullstack web application. Only basic security and authentication is implemented for the dev panel via middleware & JWT, while the rest of the backend is open to the local network.

## Folder Structure
- Handler folder for handling different data streams
    - Receiving raw bytes from the master ESP32 node, parsing it, before handing it to the appropriate handler for processing and storage in the database.
    - Handler for sending commands to the master ESP32 node via Serial communication.
    - Periodic data syncing with a cloud database (Supabase - Currently disabled) when internet connection is available.
    - Transmit messages (status, errors, warnings) to the React frontend via SSE for real time visualization.
    - Receives command from dev panel from the frontend (connect to wifi, update repo, configure running mode, etc.) and execute it on the backend.
- Lib for custom functions
    - (Extended) Kalman filter implementation on Javascript
    - KCL Corrector algorithm
- Scripts folder for handling server management (Following npm commands require you to be in the backend folder)
    - Automatic building of frontend into backend for deployment (`npm run rebuild`)
    - Automatic restart of backend after pulling repo from github (`npm run start:all`)
    - Download map tiles for offline usage (`npm run download:map`)
    - Automatic zipping and unzipping map tiles when building / downloading
- Model folder pslit into 2 file for each table
    - {__}_models.js for defining the table structure and schema
    - {__}_db.js for defining the functions to interact with the table (CRUD operations)
    - Exception: Power management requires multiple table and initialization + high speed data streaming.
    - Write Lock implemented for handling frontend download of database while still writing to db using a queue.

## Steps to Add New Data Stream Handler
<i>Note: Please check the additional notes for development on api routing (and structuring), security and test mode for future development.</i>

1. Create a new handler file in the handler > serial reader > components folder.
2. Use other handler as a template to parse, validate and queue bytes for processing and storage in the database.
3. Add to serialReader.js for the new handler to be called when receiving data from the master ESP32 node.
4. Add to the appropriate model for the new data stream to be stored in the database.
5. Add new api routes if needed for: session restore, config fetching, etc.
6. Add to lib if more complex data processing is needed (e.g. filtering, smoothing, etc.)

## Kalman Filter Implementation

Extended Kalman filter used in SoC estimation for the 2 battery bank system. The filter is implemented in Javascript and can be found in the lib folder. The filter is used to estimate the state of charge (SoC) of the battery bank based on the voltage and current measurements from the power management board.

3 filters are running simultaneously. 1 for each battery bank and 1 for the 4 Hall Effect current sensor to correct drifting using Kirchhoff's Current Law (KCL).

## Test Mode

Test mode is only implemented for power management sensor data stream. Setting test mode to true in .env or in the frontend dev panel will simulate the power management data from previous charging and discharge cycles. 

# Frontend Structure

React frontend with MUI dashboard [template](https://github.com/mui/material-ui/tree/v9.0.1/docs/data/material/getting-started/templates/dashboard).

Admin dashboard template used for data visualization and device control, with a dev panel for debugging and testing purposes (some config and mac address of microcontrollers transmited over api unencrypted as it is not a critical infrastructure).

- Each component resides in its own folder with "index.js" as the main entry point, and "styles.js" for styling. The components are organized into folders based on how they will be rendered (e.g. Dev Panel > Tabs > Database Tab > index.js).

- Server Side Event (SSE) for real time data streaming collected in "Dashboard.tsx" and passed down to the appropriate component for rendering (add more eventlisteners for different data sources).

## Steps to add new tabs:
1. Create the tab component in "components > MainBody".
2. Add on to the "mainContent" state in "Dashboard.tsx".
3. Add a new component to the main body.
4. Add to list item in "Sidebar/MenuContent.jsx" for the new tab to be rendered and selected in the sidebar.


## Dev panel control
- To access control of the ESP32 devices (strain / IMU), option will only be unlocked in the original tab after loging into the dev panel. 
- Internet: Connect to a Wi-Fi network (Does not work with some device hotspots due to mismatch of bandwidth of the device and external wifi adapter)
- Console: Direct access to the backend console for debugging and testing purposes.
- Server: Switch between Test and Normal mode, update the repo (requires internet connection), and restart the server.
- Database: View the database and export it to a CSV file for analysis.
- Logout: Logout

# Running the App

1. Copy .env-example to .env (In backend folder)
    - Please fill up the details for JWT_SECRET using an online generator
    - SUPABASE_URL and SUPABASE_KEY are optional for future cloud database sync (Implemented but disabled for now)
    - Flags are stored in .env for development purposes. Feel free to change / seperate them into a different config file / .env-flag for production use (Update the env reference accordingly).
2. `cd` into `bosun` / root folder of the repo
3. Ensure npm is installed on your system.
4. Run `npm run install:yarn`.
5. Run `npm run rebuild`.
6. Run `npm run start:all`.
   - This automatically builds the React app and runs it with Node.js.
7. Open `http://localhost:4000` in your browser.
8. Dev panel access: username=admin   password=admin

---

# Additional Notes

- A stronger computer / mini pc can be used to run the server instead for a larger vessel where more users are expected to connect to the server at the same time, or for a more complex system with more sensors and data streams and AI processing or sailing / electrical "expert" trained model can be implemented to provide real time feedback and control of the vessel.
- For production use / future development, proper security measures, such as ESP-now encryption, encrypted data (on top of current JWT authentication), and secure communication protocols (HTTPS, WSS) should be implemented to protect the system from potential attacks and unauthorized access.
- As more sensor data is added, proper routing and folder sturcture should be implemented. Currently, all api end points are dumped into index.js as there is not a lot yet. 
- Test mode is currently limited to power management data stream but should be extended to other data stream as data will be collected into the db and be downloadable for future testing.
