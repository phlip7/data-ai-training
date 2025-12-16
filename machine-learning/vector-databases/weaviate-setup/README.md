
# Environment Setup Guide

## Prerequisites

- **Conda** (Anaconda or Miniconda)
- **Docker** and Docker Compose
- **Git**

## Quick Setup

### 1. Clone the Repository

```bash
git clone https://github.com/phlip7/data-ai-training.git
cd data-ai-training
```

### 2. Create Conda Environment

```bash
# Create environment with Python 3.11
conda create -n ml python=3.11 -y

# Activate environment
conda activate ml
```

### 3. Install Python Dependencies

```bash
# Install core data science libraries
conda install numpy pandas scikit-learn matplotlib seaborn jupyter notebook -y

# Install Weaviate client (specific version)
pip install weaviate-client==4.18.1

# Install additional libraries
pip install openai python-dotenv
```

**Or use requirements.txt (if available):**
```bash
pip install -r requirements.txt
```

### 4. Setup Weaviate (Vector Database)

```bash
# Navigate to Weaviate setup folder
cd data-engineering/vector-databases/weaviate-setup

# Start Weaviate with Docker Compose
docker-compose up -d

# Verify Weaviate is running
curl http://localhost:8080/v1/.well-known/ready
```

### 5. Launch Jupyter Notebook

```bash
# From project root
conda activate ml
jupyter notebook
```

## Environment Management

### Activate Environment
```bash
conda activate ml
```

### Deactivate Environment
```bash
conda deactivate
```

### List All Environments
```bash
conda env list
```

### Update Dependencies
```bash
# Update specific package
pip install --upgrade weaviate-client

# Update all packages
pip install --upgrade -r requirements.txt
```

### Export Environment
```bash
# Export to requirements.txt
pip freeze > requirements.txt

# Export conda environment
conda env export > environment.yml
```

### Recreate Environment from Export
```bash
# From requirements.txt
pip install -r requirements.txt

# From environment.yml
conda env create -f environment.yml
```

## Docker Services

### Weaviate Commands

```bash
# Start Weaviate
docker-compose up -d

# Stop Weaviate
docker-compose down

# View logs
docker-compose logs -f weaviate

# Restart Weaviate
docker-compose restart

# Stop and remove data (reset)
docker-compose down -v
```

### Verify Docker Services
```bash
# Check running containers
docker ps

# Test Weaviate connection
curl http://localhost:8080/v1/meta
```

### install llama3 model
```bash
# pull ollama model
ollama pull llama3

## Troubleshooting

### Conda Environment Not Found
```bash
# Verify conda is installed
conda --version

# Reinitialize conda
conda init bash  # or zsh, depending on your shell
# Restart terminal
```

### Port 8080 Already in Use
```bash
# Check what's using the port
lsof -i :8080

# Kill the process or change port in docker-compose.yml
ports:
  - "8081:8080"
```

### Jupyter Not Found
```bash
conda activate ml
pip install jupyter notebook
```

### Docker Permission Denied (Linux)
```bash
sudo usermod -aG docker $USER
# Log out and back in
```

## Development Workflow

1. **Start your session:**
   ```bash
   conda activate ml
   docker-compose up -d  # If using Weaviate
   jupyter notebook
   ```

2. **End your session:**
   ```bash
   # Save your work in notebooks
   docker-compose down  # Stop services
   conda deactivate
   ```

3. **Push changes to Git:**
   ```bash
   git add .
   git commit -m "Your message"
   git push
   ```

## Project Structure

```
data-ai-training/
├── data-engineering/
│   └── vector-databases/
│       └── weaviate-setup/
│           ├── docker-compose.yml
│           ├── .env
│           └── notebooks/
├── machine-learning/
├── requirements.txt
├── .gitignore
└── README.md
```

## Additional Notes

- **API Keys**: Store sensitive keys in `.env` files (never commit these!)
- **Data Files**: Large datasets should not be committed to Git
- **Notebooks**: Clear output before committing to keep repo clean
- **Docker Volumes**: Data persists in Docker volumes even after container stops

## Useful Resources

- [Conda Documentation](https://docs.conda.io/)
- [Weaviate Documentation](https://weaviate.io/developers/weaviate)
- [Docker Documentation](https://docs.docker.com/)
- [Jupyter Documentation](https://jupyter.org/documentation)