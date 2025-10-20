# AWS App Runner Deployment Guide

Questa guida ti mostra come deployare l'app ChatKit su AWS App Runner usando Docker.

## Prerequisiti

- Account AWS attivo
- AWS CLI installato (opzionale ma consigliato)
- Docker installato localmente per test

## Opzione 1: Deploy tramite AWS Console (Più Semplice)

### Step 1: Prepara il Repository

Puoi usare:
- **GitHub** (consigliato) - App Runner può buildare direttamente da GitHub
- **Amazon ECR** - Upload manuale dell'immagine Docker

### Step 2: Deploy su AWS App Runner

1. **Vai su AWS Console → App Runner**
   - URL: https://console.aws.amazon.com/apprunner

2. **Crea un nuovo servizio**
   - Click su "Create service"

3. **Configura Source**

   **Opzione A - Da GitHub (Consigliato):**
   - Repository type: "Source code repository"
   - Connetti il tuo GitHub account
   - Seleziona il repository: `openai-chatkit-starter-app`
   - Branch: `main`
   - Deployment trigger: "Automatic" (deploy ad ogni push)

   **Opzione B - Da ECR:**
   - Repository type: "Container registry"
   - Provider: "Amazon ECR"
   - Seleziona la tua immagine

4. **Build settings** (solo per GitHub)
   - Runtime: Docker
   - Build command: (lascia vuoto, usa il Dockerfile)
   - Port: `3000`

5. **Service settings**
   - Service name: `chatkit-app` (o quello che preferisci)
   - CPU: 1 vCPU
   - Memory: 2 GB
   - Port: `3000`

6. **Environment variables** (IMPORTANTE!)

   Aggiungi queste variabili:
   ```
   OPENAI_API_KEY = sk-proj-vDuGSrFQLgb1tCKr8CQ9...
   NEXT_PUBLIC_CHATKIT_WORKFLOW_ID = wf_68f0d75a4bec8190b9633a399a1cc9a508dbfab9554bdaaf
   NODE_ENV = production
   ```

7. **Security**
   - Instance role: Usa il ruolo di default o creane uno nuovo
   - Encryption: Lascia abilitata

8. **Auto scaling**
   - Min instances: 1
   - Max instances: 5 (regola in base al traffico)

9. **Health check**
   - Path: `/`
   - Interval: 5 secondi
   - Timeout: 2 secondi

10. **Review and Create**

### Step 3: Ottieni il Dominio e Configura OpenAI

1. Dopo il deploy, App Runner ti assegnerà un dominio tipo:
   ```
   https://xxxxxxxx.us-east-1.awsapprunner.com
   ```

2. **Aggiungi questo dominio alla OpenAI Domain Allowlist:**
   - Vai su: https://platform.openai.com/settings/organization/security/domain-allowlist
   - Click "Add domain"
   - Aggiungi: `xxxxxxxx.us-east-1.awsapprunner.com`
   - Salva

3. **Testa l'applicazione** - Visita il dominio App Runner

## Opzione 2: Deploy tramite AWS CLI

### Step 1: Build e Push su ECR

```bash
# 1. Crea un repository ECR
aws ecr create-repository --repository-name chatkit-app --region us-east-1

# 2. Login su ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

# 3. Build l'immagine Docker
docker build -t chatkit-app .

# 4. Tag l'immagine
docker tag chatkit-app:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/chatkit-app:latest

# 5. Push su ECR
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/chatkit-app:latest
```

### Step 2: Crea il servizio App Runner

Crea un file `apprunner.json`:

```json
{
  "ServiceName": "chatkit-app",
  "SourceConfiguration": {
    "ImageRepository": {
      "ImageIdentifier": "<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/chatkit-app:latest",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "3000",
        "RuntimeEnvironmentVariables": {
          "NODE_ENV": "production",
          "OPENAI_API_KEY": "sk-proj-...",
          "NEXT_PUBLIC_CHATKIT_WORKFLOW_ID": "wf_..."
        }
      }
    }
  },
  "InstanceConfiguration": {
    "Cpu": "1 vCPU",
    "Memory": "2 GB"
  }
}
```

Deploy:
```bash
aws apprunner create-service --cli-input-json file://apprunner.json --region us-east-1
```

## Test Locale con Docker

Prima di deployare su AWS, testa localmente:

```bash
# 1. Build l'immagine
docker build -t chatkit-app .

# 2. Run il container
docker run -p 3000:3000 \
  -e OPENAI_API_KEY="sk-proj-..." \
  -e NEXT_PUBLIC_CHATKIT_WORKFLOW_ID="wf_..." \
  chatkit-app

# Oppure usa docker-compose
docker-compose up
```

Visita: http://localhost:3000

## Test con Docker Compose

```bash
# Start
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

## Costi Stimati AWS App Runner

- **Base**: $0.007/ora per 1 vCPU + 2GB RAM = ~$5/mese
- **Traffico**: $0.064/GB (primi 100GB gratis al mese)
- **Build**: Gratis per build da GitHub
- **Storage ECR**: $0.10/GB-mese (se usi ECR)

**Totale stimato**: $5-15/mese per uso basico

## Monitoring

1. **Logs**: AWS CloudWatch Logs (automatico)
2. **Metrics**: AWS CloudWatch Metrics
3. **Traces**: AWS X-Ray (opzionale)

Accedi ai logs da:
```
AWS Console → App Runner → [tuo servizio] → Logs
```

## Custom Domain (Opzionale)

Se hai un dominio tuo:

1. **Aggiungi custom domain in App Runner**
   - Service → Custom domains → Add domain
   - Inserisci il tuo dominio (es: chat.tuodominio.com)

2. **Configura DNS**
   - Aggiungi i record CNAME forniti da App Runner
   - Valida il dominio

3. **Aggiungi alla OpenAI Allowlist**
   - Sostituisci il dominio App Runner con il tuo custom domain

## Troubleshooting

### Build fallisce
- Controlla che `next.config.ts` abbia `output: "standalone"`
- Verifica che il Dockerfile sia nella root del progetto

### App non parte
- Controlla i logs in CloudWatch
- Verifica le environment variables
- Assicurati che la porta sia 3000

### ChatKit non si carica
- Verifica che il dominio sia nella allowlist OpenAI
- Controlla che `OPENAI_API_KEY` sia configurata correttamente
- Verifica che l'API key sia della stessa org/progetto del workflow

### 401/403 errori
- L'API key potrebbe essere scaduta o non valida
- L'API key deve avere accesso ChatKit beta
- Verifica che il workflow sia pubblicato su Agent Builder

## Rollback

Se qualcosa va storto:

```bash
# Via CLI
aws apprunner list-operations --service-arn <SERVICE_ARN>
aws apprunner start-deployment --service-arn <SERVICE_ARN> --deployment-id <OLD_DEPLOYMENT_ID>

# Via Console
App Runner → Service → Deployments → [seleziona deployment precedente] → Redeploy
```

## Comparazione: Vercel vs AWS App Runner

| Feature | Vercel | AWS App Runner |
|---------|--------|----------------|
| Setup | Più semplice | Semplice |
| Deploy time | ~1-2 min | ~3-5 min |
| Costo (basico) | Free tier generoso | ~$5-10/mese |
| Custom domain | Incluso | Incluso |
| Auto-scaling | Automatico | Configurabile |
| Logs | Dashboard Vercel | CloudWatch |
| Edge locations | Globale | Regione singola |
| Controllo | Limitato | Più controllo |

## Prossimi Passi

1. ✅ Files Docker creati
2. ⏳ Testa localmente con `docker-compose up`
3. ⏳ Deploy su AWS App Runner
4. ⏳ Aggiungi dominio alla allowlist OpenAI
5. ⏳ Testa l'app in produzione

## Supporto

- AWS App Runner docs: https://docs.aws.amazon.com/apprunner/
- OpenAI ChatKit docs: https://openai.github.io/chatkit-js/
- Issues: Controlla i logs CloudWatch per errori dettagliati
