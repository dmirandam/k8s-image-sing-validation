# k8s-image-sing-validation

Este repositorio contiene un ejemplo para construir, firmar y validar una imagen de contenedor dentro de un clúster de Kubernetes.

El flujo completo se realiza con GitHub Actions, [cosign](https://github.com/sigstore/cosign) y una política de [Kyverno](https://kyverno.io/) que rechaza aquellas imágenes que no estén firmadas correctamente.

## Componentes del repositorio

- **Dockerfile**: construye una imagen basada en `nginx`.
- **VERSION**: define el nombre y la etiqueta que se utilizarán para crear la imagen.
- **.github/workflows/pipeline.yml**: workflow de GitHub Actions que construye la imagen, la sube a un registro y la firma con cosign utilizando identidad OIDC.
- **kyverno-cluster-policy.yaml**: política de Kyverno que verifica la firma de las imágenes provenientes del registro indicado.
- **deployment.yaml**: ejemplo de despliegue de Kubernetes que usa la imagen firmada.

## ¿Cómo funciona?

1. Al ejecutarse el workflow `pipeline.yml` se lee el archivo `VERSION` para obtener el nombre y la etiqueta de la imagen.
2. La imagen se construye a partir del `Dockerfile` y se publica en el registro definido por las variables de entorno del workflow.
3. Mediante cosign se firma la imagen de forma _keyless_, aprovechando la identidad que provee GitHub OIDC. El propio workflow verifica la firma tras generarla.
4. En el clúster de Kubernetes se aplica `kyverno-cluster-policy.yaml`. Esta política comprueba que cualquier imagen de `registry.dmirandam.com/crypto/*` presente una firma válida cuyo `issuer` y `subject` coincidan con los del workflow.
5. El manifiesto `deployment.yaml` despliega un `Deployment` y un `Service` que utilizan la imagen firmada. Si la política está habilitada y la imagen no estuviera firmada, Kyverno impediría su creación.

## Uso

1. Configure en GitHub los secretos necesarios para que el workflow pueda subir imágenes a su registro (usuario, contraseña y URL).
2. Ejecute el workflow o envíe cambios al repositorio para que la imagen se construya y firme automáticamente.
3. Aplique la política de Kyverno en el clúster:
   ```bash
   kubectl apply -f kyverno-cluster-policy.yaml
   ```
4. Etiquete el _namespace_ donde quiera aplicar la validación:
   ```bash
   kubectl label namespace crypto signed-images=enforce
   ```
5. Despliegue la aplicación de ejemplo:
   ```bash
   kubectl apply -f deployment.yaml
   ```

Si la imagen ha sido firmada por el workflow indicado, el despliegue se completará con éxito; de lo contrario, la política de Kyverno lo rechazará.
