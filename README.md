# LAMP Stack with Ansible & Docker 🚀
 
## Architecture
 
```
Local Machine (Browser)
        │
        ├── localhost:8080 ──► Web Container (Apache + PHP)
        │                              │
        │                              └── connects to ──► DB Container (MySQL)
        │
        └── localhost:3306 ──► DB Container (MySQL)
 
All containers on Docker network: lamp-network
```
 
---
 
## Project Structure
 
```
lamp-ansible/
├── Dockerfile
├── docker-compose.yml
├── ansible.cfg
├── site.yml
├── ssh-setup.yml
├── inventory/
│   └── hosts.ini
└── roles/
    ├── webserver/
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   ├── templates/index.php.j2
    │   └── vars/main.yml
    └── database/
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── vars/main.yml
```
 
---
 
## Requirements
 
- Docker
- Docker Compose V2
---
 
## Step 1: Create Project Folder
 
```bash
mkdir lamp-ansible && cd lamp-ansible
```
 
---
 
## Step 2: Create Dockerfile
 
Create a file called `Dockerfile`:
 
```dockerfile
FROM ubuntu:22.04
 
ENV DEBIAN_FRONTEND=noninteractive
 
RUN apt-get update && apt-get install -y \
    openssh-server \
    python3 \
    python3-pip \
    sudo \
    curl \
    vim \
    nano \
    net-tools \
    iproute2 \
    iputils-ping \
    sshpass \
    && apt-get clean
 
RUN mkdir /var/run/sshd
 
RUN useradd -m -s /bin/bash ansible && \
    echo "ansible:ansible" | chpasswd && \
    usermod -aG sudo ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
 
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
 
EXPOSE 22
 
CMD ["/usr/sbin/sshd", "-D"]
```
 
### Dockerfile Explained
 
| Line | Description |
|------|-------------|
| `FROM ubuntu:22.04` | Base image with specific Ubuntu version |
| `ENV DEBIAN_FRONTEND=noninteractive` | Silent mode - no prompts during install |
| `apt-get update && apt-get install` | Fetch and install required packages (including nano for editing) |
| `mkdir /var/run/sshd` | Create SSH working directory |
| `useradd ansible` | Create ansible user |
| `chpasswd` | Set password to ansible |
| `NOPASSWD:ALL` | Allow sudo without password |
| `sed -i` | Enable password authentication in SSH |
| `EXPOSE 22` | Document SSH port |
| `CMD sshd -D` | Start SSH when container launches |
 
---
 
## Step 3: Create Docker Compose File
 
Create a file called `docker-compose.yml`:
 
```yaml
networks:
  lamp-network:
    driver: bridge
 
services:
 
  control:
    build: .
    container_name: ansible-control
    hostname: control
    networks:
      - lamp-network
    stdin_open: true
    tty: true
 
  webserver:
    build: .
    container_name: ansible-web
    hostname: webserver
    networks:
      - lamp-network
    ports:
      - "8080:80"
      - "2222:22"
    stdin_open: true
    tty: true
 
  database:
    build: .
    container_name: ansible-db
    hostname: database
    networks:
      - lamp-network
    ports:
      - "2233:22"
    stdin_open: true
    tty: true
```
 
### Docker Compose Explained
 
| Setting | Description |
|---------|-------------|
| `lamp-network` | Custom network so containers talk by name |
| `driver: bridge` | Private internal network between containers |
| `container_name` | Name of the container |
| `hostname` | Hostname inside the container |
| `stdin_open: true` | Keep keyboard connected (like -i flag) |
| `tty: true` | Keep screen connected (like -t flag) |
| `8080:80` | Your machine port 8080 maps to container port 80 |
| `2222:22` | Your machine port 2222 maps to container SSH port 22 |
| `2233:22` | Your machine port 2233 maps to db container SSH port 22 |
 
### Port Mapping
 
```
YOUR MACHINE                    CONTAINERS
localhost:8080  →→→→→→→→→→→   ansible-web  (Apache)
localhost:2222  →→→→→→→→→→→   ansible-web  (SSH)
localhost:2233  →→→→→→→→→→→   ansible-db   (SSH only - no direct MySQL access!)
```
 
> Note: Inside Docker network, containers talk directly using real ports (22, 3306) by container name - no port mapping needed!
 
> **Useful Docker Commands:**
> - `docker compose logs -f` → See live logs of all containers
> - `docker compose down` → Stop and remove containers and network
> - `docker compose down --rmi all` → Also remove images
> - `docker compose down --rmi all -v` → Remove everything including volumes
 
---
 
## Step 4: Build and Start Containers
 
```bash
# Build images and start all containers
docker compose up -d --build
 
# Verify all containers are running
docker ps
```
 
Expected output:
```
CONTAINER ID   NAME              STATUS
xxxx           ansible-control   Up
xxxx           ansible-web       Up
xxxx           ansible-db        Up
```
 
---
 
## Step 5: Install Ansible on Control Node
 
```bash
# Enter control container
docker exec -it ansible-control bash
 
# Switch to ansible user
su - ansible
 
# Install Ansible
sudo apt-get update
sudo apt-get install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt-get install -y ansible
 
# Verify
ansible --version
```
 
---
 
## Step 6: Create Project Structure
 
```bash
cd ~
 
# Create roles using ansible-galaxy
mkdir lamp-ansible && cd lamp-ansible
ansible-galaxy init roles/webserver
ansible-galaxy init roles/database
 
# Create inventory directory
mkdir inventory
 
# Install tree
sudo apt-get install -y tree
 
# Verify structure
tree ~/lamp-ansible
```
 
---
 
## Step 7: Create ansible.cfg
 
```bash
nano ~/lamp-ansible/ansible.cfg
```
 
```ini
[defaults]
inventory         = inventory/hosts.ini
remote_user       = ansible
host_key_checking = False
```
 
---
 
## Step 8: Create Inventory File
 
```bash
nano ~/lamp-ansible/inventory/hosts.ini
```
 
```ini
[webservers]
webserver ansible_host=webserver
 
[dbservers]
database ansible_host=database
 
[all:vars]
ansible_user=ansible
ansible_password=ansible
ansible_become=true
ansible_become_method=sudo
```
 
> Note: `ansible_password` is only needed for first time SSH setup. Remove it after SSH keys are configured.
 
---
 
## Step 9: Test Connectivity
 
```bash
cd ~/lamp-ansible
ansible all -m ping
```
 
Expected output:
```
webserver | SUCCESS => { "ping": "pong" }
database  | SUCCESS => { "ping": "pong" }
```
 
---
 
## Step 11: Create SSH Setup Playbook
 
```bash
nano ~/lamp-ansible/ssh-setup.yml
```
 
```yaml
---
- name: Setup SSH Keys
  hosts: all
  become: true
  become_method: sudo
 
  tasks:
 
    - name: Generate SSH key on control node
      ansible.builtin.openssh_keypair:
        path: /home/ansible/.ssh/id_rsa
        type: rsa
        size: 4096
        force: false
      delegate_to: localhost
      run_once: true
 
    - name: Set correct permissions on private key
      ansible.builtin.file:
        path: /home/ansible/.ssh/id_rsa
        mode: '0600'
        owner: ansible
        group: ansible
      delegate_to: localhost
      run_once: true
 
    - name: Copy public key to managed nodes
      ansible.posix.authorized_key:
        user: ansible
        state: present
        key: "{{ lookup('file', '/home/ansible/.ssh/id_rsa.pub') }}"
```
 
### SSH Setup Explained
 
| Task | Description |
|------|-------------|
| `openssh_keypair` | Generate RSA key pair on control node |
| `mode: 0600` | Private key must be readable only by owner |
| `delegate_to: localhost` | Run on control node not managed nodes |
| `run_once: true` | Generate key only once not for each host |
| `authorized_key` | Copy public key to webserver and database |
 
```bash
# Run SSH setup
ansible-playbook ssh-setup.yml
 
# Test SSH connection without password
ssh ansible@webserver "echo Web OK"
ssh ansible@database "echo DB OK"
```
 
After SSH keys work, remove password from inventory:
```ini
# Remove this line from hosts.ini
ansible_password=ansible
```
 
---
 
## Step 12: Create Webserver Role
 
### Variables
 
```bash
nano ~/lamp-ansible/roles/webserver/vars/main.yml
```
 
```yaml
---
apache_packages:
  - apache2
  - php
  - php-mysql
  - php-curl
  - php-mbstring
 
db_host: database
db_name: app_db
db_user: app_user
db_password: app_password
```
 
### Tasks
 
```bash
nano ~/lamp-ansible/roles/webserver/tasks/main.yml
```
 
```yaml
---
- name: Install Apache and PHP packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
    update_cache: true
  loop: "{{ apache_packages }}"
  notify: Restart Apache
 
- name: Start Apache service
  ansible.builtin.command:
    cmd: service apache2 start
 
- name: Set index.php as default page
  ansible.builtin.lineinfile:
    path: /etc/apache2/mods-enabled/dir.conf
    regexp: 'DirectoryIndex'
    line: 'DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm'
  notify: Restart Apache
 
- name: Deploy PHP application
  ansible.builtin.template:
    src: index.php.j2
    dest: /var/www/html/index.php
    mode: '0644'
```
 
### Handlers
 
```bash
nano ~/lamp-ansible/roles/webserver/handlers/main.yml
```
 
```yaml
---
- name: Restart Apache
  ansible.builtin.command:
    cmd: service apache2 restart
```
 
### PHP Todo App Template
 
```bash
nano ~/lamp-ansible/roles/webserver/templates/index.php.j2
```
 
```php
<?php
$host     = "{{ db_host }}";
$dbname   = "{{ db_name }}";
$username = "{{ db_user }}";
$password = "{{ db_password }}";
 
try {
    $conn = new PDO("mysql:host=$host;dbname=$dbname", $username, $password);
    $conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
    // Create table if not exists
    $conn->exec("CREATE TABLE IF NOT EXISTS todos (
        id INT AUTO_INCREMENT PRIMARY KEY,
        title VARCHAR(255) NOT NULL,
        status ENUM('pending','done') DEFAULT 'pending',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )");
 
    // Add new todo
    if (isset($_POST['add'])) {
        $stmt = $conn->prepare("INSERT INTO todos (title) VALUES (:title)");
        $stmt->execute([':title' => $_POST['title']]);
    }
 
    // Mark as done
    if (isset($_POST['done'])) {
        $stmt = $conn->prepare("UPDATE todos SET status='done' WHERE id=:id");
        $stmt->execute([':id' => $_POST['id']]);
    }
 
    // Delete todo
    if (isset($_POST['delete'])) {
        $stmt = $conn->prepare("DELETE FROM todos WHERE id=:id");
        $stmt->execute([':id' => $_POST['id']]);
    }
 
    // Get all todos
    $todos = $conn->query("SELECT * FROM todos ORDER BY created_at DESC")->fetchAll();
 
} catch(PDOException $e) {
    die("<h2>Connection Failed: " . $e->getMessage() . "</h2>");
}
?>
 
<!DOCTYPE html>
<html>
<head>
    <title>Todo List - LAMP Ansible</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: Arial; background: #f0f2f5; min-height: 100vh; padding: 40px 20px; }
        .container { max-width: 700px; margin: 0 auto; }
        h1 { text-align: center; color: #333; margin-bottom: 30px; }
        .add-form { background: white; padding: 20px; border-radius: 10px;
                    box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 20px; display: flex; gap: 10px; }
        .add-form input { flex: 1; padding: 12px; border: 2px solid #ddd;
                          border-radius: 8px; font-size: 16px; outline: none; }
        .add-form input:focus { border-color: #4CAF50; }
        .btn { padding: 12px 20px; border: none; border-radius: 8px; cursor: pointer; font-size: 14px; }
        .btn-add { background: #4CAF50; color: white; }
        .btn-done { background: #2196F3; color: white; }
        .btn-delete { background: #f44336; color: white; }
        .todo-item { background: white; padding: 15px 20px; border-radius: 10px;
                     box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 10px;
                     display: flex; align-items: center; gap: 10px; }
        .todo-item.done { opacity: 0.6; }
        .todo-title { flex: 1; font-size: 16px; color: #333; }
        .todo-title.done { text-decoration: line-through; color: #999; }
        .badge { padding: 4px 10px; border-radius: 20px; font-size: 12px; }
        .badge-pending { background: #fff3cd; color: #856404; }
        .badge-done { background: #d4edda; color: #155724; }
        .empty { text-align: center; color: #999; padding: 40px; background: white;
                 border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .stats { display: flex; gap: 10px; margin-bottom: 20px; }
        .stat { flex: 1; background: white; padding: 15px; border-radius: 10px;
                box-shadow: 0 2px 10px rgba(0,0,0,0.1); text-align: center; }
        .stat-number { font-size: 28px; font-weight: bold; color: #4CAF50; }
        .stat-label { font-size: 12px; color: #999; }
    </style>
</head>
<body>
<div class="container">
    <h1>📝 Todo List - LAMP Stack with Ansible</h1>
 
    <?php
        $total   = count($todos);
        $done    = count(array_filter($todos, fn($t) => $t['status'] == 'done'));
        $pending = $total - $done;
    ?>
    <div class="stats">
        <div class="stat">
            <div class="stat-number"><?= $total ?></div>
            <div class="stat-label">Total</div>
        </div>
        <div class="stat">
            <div class="stat-number" style="color:#2196F3"><?= $pending ?></div>
            <div class="stat-label">Pending</div>
        </div>
        <div class="stat">
            <div class="stat-number" style="color:#4CAF50"><?= $done ?></div>
            <div class="stat-label">Done</div>
        </div>
    </div>
 
    <form method="POST" class="add-form">
        <input type="text" name="title" placeholder="Add a new todo..." required>
        <button type="submit" name="add" class="btn btn-add">➕ Add</button>
    </form>
 
    <?php if (empty($todos)): ?>
        <div class="empty">
            <h3>No todos yet!</h3>
            <p>Add your first todo above 👆</p>
        </div>
    <?php else: ?>
        <?php foreach ($todos as $todo): ?>
        <div class="todo-item <?= $todo['status'] == 'done' ? 'done' : '' ?>">
            <span class="todo-title <?= $todo['status'] == 'done' ? 'done' : '' ?>">
                <?= htmlspecialchars($todo['title']) ?>
            </span>
            <span class="badge <?= $todo['status'] == 'done' ? 'badge-done' : 'badge-pending' ?>">
                <?= $todo['status'] ?>
            </span>
            <?php if ($todo['status'] == 'pending'): ?>
            <form method="POST">
                <input type="hidden" name="id" value="<?= $todo['id'] ?>">
                <button type="submit" name="done" class="btn btn-done">✅ Done</button>
            </form>
            <?php endif; ?>
            <form method="POST">
                <input type="hidden" name="id" value="<?= $todo['id'] ?>">
                <button type="submit" name="delete" class="btn btn-delete">🗑️ Delete</button>
            </form>
        </div>
        <?php endforeach; ?>
    <?php endif; ?>
</div>
</body>
</html>
```
 
---
 
## Step 13: Create Database Role
 
### Variables
 
```bash
nano ~/lamp-ansible/roles/database/vars/main.yml
```
 
```yaml
---
db_packages:
  - mysql-server
  - python3-mysqldb
 
db_name: app_db
db_user: app_user
db_password: app_password
db_root_password: rootpassword
```
 
### Tasks
 
```bash
nano ~/lamp-ansible/roles/database/tasks/main.yml
```
 
```yaml
---
- name: Install MySQL packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
    update_cache: true
  loop: "{{ db_packages }}"
 
- name: Start MySQL service
  ansible.builtin.command:
    cmd: service mysql start
 
- name: Allow MySQL to listen on all interfaces
  ansible.builtin.lineinfile:
    path: /etc/mysql/mysql.conf.d/mysqld.cnf
    regexp: '^bind-address'
    line: 'bind-address = 0.0.0.0'
  notify: Restart MySQL
 
- name: Force handler to run now
  ansible.builtin.meta: flush_handlers
 
- name: Create database
  community.mysql.mysql_db:
    name: "{{ db_name }}"
    state: present
 
- name: Create database user
  community.mysql.mysql_user:
    name: "{{ db_user }}"
    password: "{{ db_password }}"
    priv: "{{ db_name }}.*:ALL"
    host: "%"
    state: present
```
 
### Handlers
 
```bash
nano ~/lamp-ansible/roles/database/handlers/main.yml
```
 
```yaml
---
- name: Restart MySQL
  ansible.builtin.command:
    cmd: service mysql restart
```
 
---
 
## Step 14: Create Master Playbook
 
```bash
nano ~/lamp-ansible/site.yml
```
 
```yaml
---
- name: Configure Web Server
  hosts: webservers
  become: true
  become_method: sudo
  roles:
    - webserver
 
- name: Configure Database Server
  hosts: dbservers
  become: true
  become_method: sudo
  roles:
    - database
```
 
---
 
## Step 15: Run Full Playbook
 
```bash
# Syntax check first
ansible-playbook site.yml --syntax-check
 
# Dry run
ansible-playbook site.yml --check
 
# Run for real
ansible-playbook site.yml
```
 
---
 
## Step 16: Verify Everything
 
```bash
# Check Apache running
ansible webservers -m command -a "service apache2 status"
 
# Check MySQL running
ansible dbservers -m command -a "service mysql status"
 
# Check database created
ansible dbservers -m command -a "mysql -u root -e 'show databases;'"
 
# Check user created
ansible dbservers -m command -a "mysql -u root -e 'select user, host from mysql.user;'"
 
# Check PHP file deployed
ansible webservers -m command -a "ls /var/www/html/"
```
 
---
 
## Step 17: Access Todo App
 
Open your browser:
```
http://localhost:8080
```
 
 
 
