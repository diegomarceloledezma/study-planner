# Configuración de GitHub Actions para Deploy en AWS EC2

Este proyecto usa GitHub Actions para automatizar el build y deploy de la aplicación en AWS EC2.

## Prerequisitos en AWS

### 1. Crear un repositorio ECR
```bash
aws ecr create-repository --repository-name studyplanner2025 --region us-east-1
```

### 2. Configurar EC2
- Instalar Docker en tu instancia EC2:
```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

- Instalar AWS CLI:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### 3. Configurar Security Group
Asegúrate de que tu EC2 tenga estas reglas en el Security Group:
- Puerto 22 (SSH) - desde tu IP o GitHub Actions
- Puerto 5000 (Flask) - desde 0.0.0.0/0 (o tu IP específica)

## Secrets de GitHub

Debes agregar estos secrets en tu repositorio de GitHub (Settings > Secrets and variables > Actions):

### Secrets Requeridos:

1. **AWS_ACCESS_KEY_ID**
   - Tu AWS Access Key ID con permisos para ECR

2. **AWS_SECRET_ACCESS_KEY**
   - Tu AWS Secret Access Key

3. **EC2_HOST**
   - La IP pública o DNS de tu instancia EC2
   - Ejemplo: `ec2-12-34-56-78.compute-1.amazonaws.com`

4. **EC2_USER**
   - Usuario SSH de tu EC2 (normalmente `ec2-user` para Amazon Linux)

5. **EC2_SSH_KEY**
   - Tu clave privada SSH completa para conectarte a EC2
   - Copia todo el contenido de tu archivo .pem incluyendo:
     ```
     -----BEGIN RSA PRIVATE KEY-----
     [contenido de la clave]
     -----END RSA PRIVATE KEY-----
     ```

## Cómo Agregar los Secrets

1. Ve a tu repositorio en GitHub
2. Click en **Settings** > **Secrets and variables** > **Actions**
3. Click en **New repository secret**
4. Agrega cada secret con su nombre y valor correspondiente

## Permisos IAM Necesarios

Tu usuario de AWS debe tener estas políticas:
- `AmazonEC2ContainerRegistryFullAccess` (para ECR)
- O crear una política personalizada:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

## Cómo Funciona el Pipeline

1. **Trigger**: Se ejecuta al hacer push a la rama `main` o manualmente
2. **Build**: Construye la imagen Docker usando Alpine Linux
3. **Push**: Sube la imagen a Amazon ECR
4. **Deploy**: Se conecta por SSH a tu EC2 y:
   - Descarga la nueva imagen desde ECR
   - Detiene el contenedor anterior
   - Inicia el nuevo contenedor en el puerto 5000
   - Limpia imágenes antiguas

## Ejecutar el Pipeline

Simplemente haz push a la rama main:
```bash
git add .
git commit -m "Deploy to EC2"
git push origin main
```

O ejecuta manualmente desde GitHub:
- Ve a **Actions**
- Selecciona **Deploy to AWS EC2**
- Click en **Run workflow**

## Verificar el Deploy

Después del deploy, tu aplicación estará disponible en:
```
http://TU_EC2_HOST:5000
```

## Troubleshooting

Si el pipeline falla:
1. Verifica que todos los secrets estén configurados correctamente
2. Revisa los logs en la pestaña Actions de GitHub
3. Verifica que el Security Group permita el puerto 5000
4. Asegúrate de que Docker esté corriendo en EC2: `sudo systemctl status docker`
