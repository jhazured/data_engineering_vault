
**Tags:** #goanywhere #mft #sftp #xml #configuration #file-transfer #network-share

## Overview

This document provides an updated example of the GoAnywhere MFT XML configuration, where the source folder is an SFTP server, and the destination folder is a network-shared drive (on your local or network file system).

## Scenario Configuration

**Source Directory (SFTP Server):** `/remote/folder/` (on the SFTP server managed by GoAnywhere MFT)  
**Destination Directory (Network Share):** `\\network-share\path\to\destination\` (local or network shared drive)  
**Task:** Move files from the remote SFTP folder to a network shared location

---

## Sample XML Configuration

### Complete XML Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Project>
    <Name>MoveFilesSFTPtoNetworkShare</Name>
    <Description>Automated task to move files from an SFTP server to a network shared drive</Description>
    <Steps>
        <!-- Step 1: Move Files from SFTP to Network Share -->
        <Step>
            <Type>SFTP</Type> <!-- SFTP transfer step -->
            <StepName>Move Files from SFTP</StepName>
            <Action>download</Action> <!-- Download files from SFTP server -->
            <Host>sftp.example.com</Host> <!-- Your SFTP server host -->
            <Port>22</Port> <!-- Default SFTP port -->
            <Username>your_sftp_user</Username> <!-- SFTP username -->
            <Password>your_sftp_password</Password> <!-- SFTP password -->
            <AuthenticationMethod>password</AuthenticationMethod> <!-- Method of authentication (password or sshkey) -->
            <RemoteDirectory>/remote/folder/</RemoteDirectory> <!-- Source SFTP directory -->
            <LocalDirectory>\\network-share\path\to\destination\</LocalDirectory> <!-- Network shared directory as destination -->
            <FilePattern>*.*</FilePattern> <!-- Transfer all files -->
            <Overwrite>no</Overwrite> <!-- Do not overwrite existing files -->
            <Timeout>600</Timeout> <!-- Timeout for the operation in seconds -->
            <Retries>3</Retries> <!-- Number of retries on failure -->
        </Step>

        <!-- Optional: Step 2, Send notification after successful move -->
        <Step>
            <Type>Notification</Type>
            <StepName>Send Notification</StepName>
            <Subject>File Move Completed</Subject>
            <Body>Files have been successfully moved from the SFTP server to the network share.</Body>
            <RecipientEmail>admin@example.com</RecipientEmail> <!-- Email recipient -->
        </Step>
    </Steps>

    <!-- Optional: Actions on success and failure -->
    <PostSuccessAction>
        <Type>LogMessage</Type>
        <Message>File move operation from SFTP to network share completed successfully.</Message>
    </PostSuccessAction>

    <PostFailureAction>
        <Type>LogMessage</Type>
        <Message>File move operation from SFTP to network share failed. Please check logs for details.</Message>
    </PostFailureAction>
</Project>
```

---

## Key Configuration Elements

### 1. Source Directory Configuration

- **`<Host>`:** `sftp.example.com` — The hostname or IP address of the SFTP server
- **`<Port>`:** `22` — Default SFTP port (can be adjusted if different)
- **`<Username>`:** `your_sftp_user` — Your SFTP username
- **`<Password>`:** `your_sftp_password` — Your SFTP password (or you can use an SSH key, if preferred)
- **`<RemoteDirectory>`:** `/remote/folder/` — The path to the source folder on the SFTP server where the files are located

### 2. Destination Directory Configuration

- **`<LocalDirectory>`:** `\\network-share\path\to\destination\` — This should point to the network share location where the files will be moved

### 3. File Transfer Settings

- The SFTP action is set to `download`, which means files will be downloaded from the remote SFTP server to the local network shared folder
- **`<FilePattern>`:** `*.*` — This will select all files in the source folder for download. You can filter this with more specific patterns (e.g., `*.txt` for text files)

### 4. Overwrite Configuration

- **`<Overwrite>`** is set to `no` to prevent overwriting existing files on the network share. You can change this to `yes` if overwriting is allowed

### 5. Network Share Considerations

- Make sure the network share is properly mapped and accessible by GoAnywhere MFT
- You may need to configure the credentials or network path in GoAnywhere's settings (like using a mapped network drive or UNC path)
- If GoAnywhere is running on a Linux server, you'll need to mount the shared folder properly (using CIFS/SMB)

### 6. Optional Components

- **Notification Step:** This step sends an email once the file transfer is complete. You can customize the subject, body, and recipient email as needed
- **Post-Success and Post-Failure Actions:** Logs success or failure messages for auditing purposes

---

## Example Use Case: Windows Server Setup

For a Windows server running GoAnywhere MFT, moving files from an SFTP server to a network shared folder like `\\Server\Backup\`:

```xml
<Step>
    <Type>SFTP</Type> <!-- Use SFTP for remote file transfer -->
    <StepName>Move Files from SFTP to Network Share</StepName>
    <Action>download</Action> <!-- Download files from the SFTP server -->
    <Host>sftp.example.com</Host> <!-- SFTP server address -->
    <Port>22</Port> <!-- SFTP port -->
    <Username>your_sftp_user</Username> <!-- SFTP login username -->
    <Password>your_sftp_password</Password> <!-- SFTP login password -->
    <AuthenticationMethod>password</AuthenticationMethod> <!-- Choose either 'password' or 'sshkey' -->
    <RemoteDirectory>/remote/folder/</RemoteDirectory> <!-- Remote directory on SFTP server -->
    <LocalDirectory>\\Server\Backup\</LocalDirectory> <!-- Network share destination -->
    <FilePattern>*.*</FilePattern> <!-- Transfer all files -->
    <Overwrite>no</Overwrite> <!-- Don't overwrite existing files -->
    <Timeout>600</Timeout> <!-- Timeout in seconds -->
    <Retries>3</Retries> <!-- Retries on failure -->
</Step>
```

### Configuration Details

- The SFTP server is at `sftp.example.com` (replace with your actual host)
- Files will be downloaded from `/remote/folder/` to the network share located at `\\Server\Backup\`

> [!important] Make sure that GoAnywhere MFT has the necessary permissions to access the network share, and that the destination folder is properly mapped or accessible.

---

## Network Share Configuration Guidelines

### Windows Environment

- Make sure the network share is mapped properly, or provide the UNC path (`\\server\share\path`)

### Linux Environment

- Mount the network share using the appropriate protocol (e.g., CIFS/SMB) before referencing the path in the GoAnywhere project

### Additional Considerations

- You can configure GoAnywhere to use credentials to access the network share, if necessary
- If you encounter any issues with network share access, check the GoAnywhere documentation or server permissions

---