# 02. Enable Google Ads API and create OAuth credentials

Goal: enable the Google Ads API in the project and create an OAuth client the MCP server can use to authenticate.

## Enable the API

Went to APIs and Services, Library, searched Google Ads API.
Clicked Enable. Status changed to Enabled for service googleads.googleapis.com.

## OAuth consent screen

Went to Google Auth Platform, Overview, Get started.
App name: Google Ads MCP.
User support email: <your-email>.
Audience: Internal (restricted to your Google Workspace organization, no Google verification needed). Use External instead if you are not on a Google Workspace org.
Contact email: <your-email>.
Accepted the Google API Services User Data Policy.
Clicked Create. Got confirmation OAuth configuration created.

## OAuth client

Went to Clients, Create client.
Application type: Desktop app, since the MCP server runs locally rather than as a web service.
Name: <desktop-client-name>.
Clicked Create. Google generated a Client ID and Client secret.

## Notes

Client ID and secret are not stored in this repo, and neither is any personal email or account info. They live only in the Google Cloud console and, once downloaded, in a local credentials file on the machine running the MCP server, excluded from git via .gitignore. If you are following this as a guide, substitute your own email and organization wherever you see a placeholder like <your-email>.
