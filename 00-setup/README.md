# Fase 0: Inicialización del Sustrato (Resource Plane)

---

## 0.1 ¿Qué construimos en esta fase?

Levantamos un **clúster de Kubernetes efímero** usando `kind`. Este clúster actúa como el *Resource Plane* donde Microcks residirá, simulando el entorno de una plataforma real sin necesidad de infraestructura en la nube.

---

## 0.2 Creación del Clúster Kind

### Paso 1: Crear el clúster local

```bash
# Crear un clúster Kind con nombre api-platform
kind create cluster --name api-platform

# Verificar que el contexto de kubectl apunta al nuevo clúster
kubectl config current-context
# Debe mostrar: kind-api-platform
```

### Paso 2: Verificar el clúster

```bash
# Ver los nodos del clúster
kubectl get nodes

# Ver los namespaces existentes
kubectl get namespaces
```

Deberías ver algo similar a:

```
NAME                 STATUS   ROLES           AGE
api-platform-control-plane   Ready    control-plane   30s
```

---

## 0.3 Preparar los namespaces

```bash
# Crear el namespace donde vivirá Microcks
kubectl create namespace microcks

# Verificar
kubectl get namespaces | grep microcks
```

---

*Continúa con → [Fase 1: Microcks](../01-microcks/README.md)*
