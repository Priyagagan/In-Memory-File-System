import os
import json

class InMemoryFileSystem:
    def __init__(self):
        self.root = self.create_directory("/")
        self.current_directory = self.root

    def create_directory(self, path):
        return {'type': 'directory', 'name': os.path.basename(path), 'content': {}}

    def create_file(self, path):
        return {'type': 'file', 'name': os.path.basename(path), 'content': ''}

    def navigate_path(self, path):
        if path.startswith('/'):
            current_directory = self.root
            path = path[1:]
        else:
            current_directory = self.current_directory

        for part in path.split('/'):
            if part == '..':
                current_directory = current_directory.get('parent', current_directory)
            elif part and part != '.':
                current_directory = current_directory['content'].get(part, None)

            if current_directory is None or current_directory['type'] == 'file':
                return None

        return current_directory

    def mkdir(self, path):
        if path == '/':
            return "Cannot create root directory."

        directory_name = os.path.basename(path)
        new_directory = self.create_directory(path)
        new_directory['parent'] = self.current_directory
        self.current_directory['content'][directory_name] = new_directory
        return None

    def cd(self, path):
        new_directory = self.navigate_path(path)
        if new_directory:
            self.current_directory = new_directory
            return None
        else:
            return "Directory not found."

    def ls(self, path='.'):
        target_directory = self.navigate_path(path)
        if target_directory:
            return ', '.join(target_directory['content'].keys())
        else:
            return "Directory not found."

    def grep(self, pattern, path):
        file = self.navigate_path(path)
        if file and file['type'] == 'file':
            return pattern in file['content']
        else:
            return "File not found."

    def cat(self, path):
        file = self.navigate_path(path)
        if file and file['type'] == 'file':
            return file['content']
        else:
            return "File not found."

    def touch(self, path):
        if self.navigate_path(path):
            return "File already exists."
        else:
            file_name = os.path.basename(path)
            new_file = self.create_file(path)
            new_file['parent'] = self.current_directory
            self.current_directory['content'][file_name] = new_file
            return None

    def echo(self, content, path):
        file = self.navigate_path(path)
        if file and file['type'] == 'file':
            file['content'] = content
            return None
        else:
            return "File not found."

    def mv(self, source, destination):
        source_item = self.navigate_path(source)
        destination_directory = self.navigate_path(destination)
        if source_item and destination_directory and destination_directory['type'] == 'directory':
            del source_item['parent']['content'][source_item['name']]
            source_item['parent'] = destination_directory
            destination_directory['content'][source_item['name']] = source_item
            return None
        else:
            return "Invalid source or destination."

    def cp(self, source, destination):
        source_item = self.navigate_path(source)
        destination_directory = self.navigate_path(destination)
        if source_item and destination_directory and destination_directory['type'] == 'directory':
            if source_item['type'] == 'file':
                new_file = self.create_file(destination + '/' + source_item['name'])
                new_file['content'] = source_item['content']
                new_file['parent'] = destination_directory
                destination_directory['content'][source_item['name']] = new_file
            elif source_item['type'] == 'directory':
                new_directory = self.create_directory(destination + '/' + source_item['name'])
                new_directory['parent'] = destination_directory
                destination_directory['content'][source_item['name']] = new_directory
                self.copy_directory(source_item, new_directory)
            return None
        else:
            return "Invalid source or destination."

    def rm(self, path):
        item = self.navigate_path(path)
        if item:
            del item['parent']['content'][item['name']]
            return None
        else:
            return "File or directory not found."

    def copy_directory(self, source, destination):
        for name, item in source['content'].items():
            if item['type'] == 'file':
                new_file = self.create_file(destination['name'] + '/' + name)
                new_file['content'] = item['content']
                new_file['parent'] = destination
                destination['content'][name] = new_file
            elif item['type'] == 'directory':
                new_directory = self.create_directory(destination['name'] + '/' + name)
                new_directory['parent'] = destination
                destination['content'][name] = new_directory
                self.copy_directory(item, new_directory)

def save_state(file_system, path):
    with open(path, 'w') as file:
        json.dump(file_system.root, file)

def load_state(path):
    with open(path, 'r') as file:
        data = json.load(file)
        file_system = InMemoryFileSystem()
        file_system.root = data
        return file_system

def main():
    file_system = InMemoryFileSystem()

    while True:
        command = input("$ ").strip()

        if command.lower() == "exit":
            break

        if command.startswith("python script.py"):
            args = eval(command[len("python script.py"):])
            if 'save_state' in args and args['save_state'] == 'true':
                save_state(file_system, args['path'])
            elif 'load_state' in args and args['load_state'] == 'true':
                file_system = load_state(args['path'])
            continue

        parts = command.split(' ', 1)
        operation = parts[0]

        if operation == 'mkdir':
            result = file_system.mkdir(parts[1])
        elif operation == 'cd':
            result = file_system.cd(parts[1])
        elif operation == 'ls':
            result = file_system.ls(parts[1]) if len(parts) > 1 else file_system.ls()
        elif operation == 'grep':
            result = file_system.grep(*parts[1].split(' ', 1))
        elif operation == 'cat':
            result = file_system.cat(parts[1])
        elif operation == 'touch':
            result = file_system.touch(parts[1])
        elif operation == 'echo':
            result = file_system.echo(*parts[1].split(' ', 1))
        elif operation == 'mv':
            result = file_system.mv(*parts[1].split(' ', 1))
        elif operation == 'cp':
            result = file_system.cp(*parts[1].split(' ', 1))
        elif operation == 'rm':
            result = file_system.rm(parts[1])
        else:
            result = "Invalid command."

        if result:
            print(result)

if __name__ == "__main__" :
	main()
