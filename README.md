## Removes files and folders by moving them to Trash, with automatic versioning if an item with the same name already exists

<br>

This zsh/bash script works in macOS Tahoe and overrides the ```rm``` terminal command with ``` alias rm='rm_with_trash'```, ensuring files are moved to the Trash instead of being permanently deleted.

<img width="142" height="184" alt="image" src="https://github.com/user-attachments/assets/1f5d95a3-be1a-45b0-9cc7-9d6259beb8be" />
<br><br>

These commands were executed to produce the output shown in the image:

```bash
cd ~

mkdir folder1; cd folder1; mkdir folder2; cd folder2; touch Test-file; cd ~
rm -r folder 1

mkdir folder1; cd folder1; mkdir folder2; cd folder2; touch Test-file; cd ~
rm -r folder 1

mkdir folder1; cd folder1; mkdir folder2; cd folder2; touch Test-file; cd ~
rm -r folder 1
```

<br>

### How to use it
---

#### In zsh
> 1. Open the terminal
> 2. Copy the script once at the end of the ``` ~/.zshrc ``` file
> 3. Run the following command once: ``` source ~/.zshrc ```
> 4. Create an example folder, delete it and review the Trash
> 5. Create again the same folder, delete it, and review the Trash. Do this many times

#### In bash
> 1. Open the terminal
> 2. Copy the script once at the end of the ``` ~/.bashrc ``` file
> 3. Run the following command once: ``` source ~/.bashrc ```
> 4. Create an example folder, delete it and review the Trash
> 5. Create again the same folder, delete it, and review the Trash. Do this many times

<br>

```bash
rm_with_trash() {
  local recursive=false
  local force=false
  local prompt_mode=false
  local prompt_once=false
  local verbose=false
  local delete_dirs=false
  local items=()
  local file_count=0
  local confirmed=false
  
  # Parse flags
  for arg in "$@"; do
    if [[ "$arg" == -* ]] && [[ "$arg" != "--" ]]; then
      # Process concatenated flags
      local flags="${arg:1}"
      for ((i=0; i<${#flags}; i++)); do
        local flag="${flags:$i:1}"
        case "$flag" in
          r|R)
            recursive=true
            delete_dirs=true
            ;;
          d)
            delete_dirs=true
            ;;
          f)
            force=true
            prompt_mode=false
            prompt_once=false
            ;;
          i)
            prompt_mode=true
            force=false
            prompt_once=false
            ;;
          I)
            prompt_once=true
            force=false
            prompt_mode=false
            ;;
          v)
            verbose=true
            ;;
          x)
            echo "rm: Option -$flag not implemented" >&2
            return 1
            ;;
          W)
            echo "rm: Option -$flag  not implemented" >&2
            return 1
            ;;
          P)
            # -P flag does nothing, kept for backwards compatibility
            ;;
          *)
            echo "rm: Unknown option: -$flag" >&2
            return 1
            ;;
        esac
      done
    elif [[ "$arg" == "--" ]]; then
      # Double dash - stop processing flags, rest are files
      shift
      items+=("$@")
      break
    else
      # Regular file argument
      items+=("$arg")
    fi
  done
  
  # Check if any items provided
  if [ ${#items[@]} -eq 0 ]; then
    echo "rm: Missing operand" >&2
    return 1
  fi
  
  # Count items for -I flag
  file_count=${#items[@]}
  
  # Determine if we should prompt once
  local should_prompt_once=false
  if [ "$prompt_once" = true ] && [ $file_count -gt 3 ]; then
    should_prompt_once=true
  fi
  
  # Prompt once if needed
  if [ "$should_prompt_once" = true ]; then
    echo -n "Delete $file_count files? "
    read -r response
    if [[ ! $response =~ ^[Yy] ]]; then
      return 0
    fi
  fi
  
  # Process each item
  for item in "${items[@]}"; do
    # Validate special paths
    if [ "$item" = "/" ] || [ "$item" = "." ] || [ "$item" = ".." ]; then
      echo -e "\nWarning: Can not delete '$item'" >&2
      continue
    fi
    
    if [ ! -e "$item" ] && [ ! -L "$item" ]; then
      echo -e "\nWarning: Can not delete '$item'. No such file or folder" >&2
      # With -f flag, continue processing other files without returning error
      continue
    fi
    
    # Check if item is a folder
    if [ -d "$item" ]; then
      if [ "$recursive" = false ] && [ "$delete_dirs" = false ]; then
        echo -e "\nWarning: Can not delete '$item'. Is a folder" >&2
        continue
      fi
      
      # If -d flag is used without -r/-R, only delete empty directories
      if [ "$delete_dirs" = true ] && [ "$recursive" = false ]; then
        # Check if folder is empty
        if [ -n "$(find "$item" -maxdepth 1 -type f -o -type d -not -name "$item")" ]; then
          echo -e "\nWarning: Can not delete folder '$item'. It is not empty" >&2
          continue
        fi
      fi
      
      # Prompt if -i flag is used or -I with directories
      if [ "$prompt_mode" = true ]; then
        echo -n "Do you confirm the deletion of folder '$item'? "
        read -r response
        if [[ ! $response =~ ^[Yy] ]]; then
          continue
        fi
      elif [ "$prompt_once" = true ] && [ "$confirmed" = false ]; then
        echo -n "Do you confirm the deletion of folder '$item'? "
        read -r response
        if [[ ! $response =~ ^[Yy] ]]; then
          continue
        fi
        confirmed=true
      fi
      
      # Get the base name of the item
      local dir_basename=$(basename "$item")
      local dir_trash_path="$HOME/.Trash/$dir_basename"
      local dir_final_name="$dir_basename"
      local dir_counter=1
      
      # Check if file already exists in trash and version it     
      while [ -e "$dir_trash_path" ]; do
        if [[ "$dir_basename" == *.* ]]; then
          local dir_name="${dir_basename%.*}"
          local dir_ext=".${dir_basename##*.}"
          dir_final_name="${dir_name} [v$dir_counter]${dir_ext}"
        else
          dir_final_name="${dir_basename} [v$dir_counter]"
        fi
        dir_trash_path="$HOME/.Trash/$dir_final_name"
        ((dir_counter++))
      done
      
      # Move folder to trash
      if cp -R "$item" "$dir_trash_path" 2>/dev/null; then
        if [ "$verbose" = true ]; then
          # Show all files and directories being deleted in reverse order (deepest first)
          find "$item" -print | sort -r
        fi
        if ! /bin/rm -rf "$item" 2>/dev/null; then
          echo -e "\nWarning: Failed to delete folder '$item'" >&2
          # If failed to delete original, delete the copied files from trash
          if /bin/rm -rf "$dir_final_name" 2>/dev/null; then
            echo -e "\nWarning: Folder $dir_final_name was deleted from the trash due to an error while deleting original folder"
          else
            echo -e "\nWarning: Failed to delete the folder $dir_final_name from the trash due to an error removing the original folder" >&2
          fi
        else
          echo -e "\nFolder succesfully deleted" >&2
        fi
      else
        echo -e "\nWarning: Failed to copy folder to trash before deleting it" >&2
      fi
    else
      # It's a file
      if [ "$prompt_mode" = true ]; then
        echo -n "Do you confirm the deletion of file '$item'? "
        read -r response
        if [[ ! $response =~ ^[Yy] ]]; then
          continue
        fi
      fi
      
      # Get the base name of the item
      local basename=$(basename "$item")
      local trash_path="$HOME/.Trash/$basename"
      local final_name="$basename"
      local counter=1
      
      # Check if file already exists in trash and version it     
      while [ -e "$trash_path" ]; do
        if [[ "$basename" == *.* ]]; then
          local name="${basename%.*}"
          local ext=".${basename##*.}"
          final_name="${name} [v$counter])${ext}"
        else
          final_name="${basename} [v$counter]"
        fi
        trash_path="$HOME/.Trash/$final_name"
        ((counter++))
      done

      
      # Copy file to trash first, then delete original
      if cp "$item" "$trash_path" 2>/dev/null; then
        if [ "$verbose" = true ]; then
          echo "$item"
        fi
        if ! /bin/rm -f "$item" 2>/dev/null; then
          echo -e "\nWarning: Failed to delete file '$item'" >&2
          # If failed to delete original, delete the copied file from trash
          if /bin/rm -f "$final_name" 2>/dev/null; then
            echo -e "\nWarning: File $final_name was deleted from the trash due to an error while deleting the original file"
          else
            echo -e "\nWarning: Failed to delete the file $final_name from the trash due to an error removing the original file" >&2
          fi
        else
         echo -e "\nFile succesfully deleted" >&2
        fi
      else
        echo -e "\nWarning: Failed to copy file to trash before deleting it" >&2
      fi
    fi
  done
}

alias rm='rm_with_trash'

# Usage examples:
# rm file.txt                    # Deletes without prompt
# rm -i file.txt                 # Prompts before deleting each file
# rm -I file1 file2 file3 file4  # Prompts once for multiple files
# rm -f file.txt                 # Force delete without prompt
# rm -v file.txt                 # Verbose mode, shows files as deleted
# rm -r folder/                  # Deletes folder recursively
# rm -ri folder/                 # Prompts before deleting folder
# rm -rf folder/                 # Force deleting folder without prompt
# rm -d folder/                  # Delete empty folders
# rm file1.txt file2.txt         # Delete both files
# rm -i *.log                    # Prompts for each .log file
# Result: file.txt, file (1).txt, file (2).txt when deleted multiple times
```

<br>

### In VS Code with AI Coding Agent

<br>

> Add the following configuration to the end of `settings.json`. This file can be accessed via Cmd-Shift-P and selecting `Preferences: Open User Settings (JSON)`

#### For zsh

```json
  "terminal.integrated.defaultProfile.osx": "zsh",
  "terminal.integrated.profiles.osx": {
    "zsh": {
      "path": "/bin/zsh"      
    }
  }
```

#### For bash

```json
  "terminal.integrated.defaultProfile.osx": "bash",
  "terminal.integrated.profiles.osx": {
    "bash": {
      "path": "/bin/bash",
      "args": ["-i"]
    }
  }
```

