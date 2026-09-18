# ADR 002 — AUTH_USER_MODEL Propio con PK UUID

**Estado:** Aceptada · Septiembre 2026

**Contexto**

Sistema multi-tenant con un esquema de BD por organización. Cada organización tiene su tabla `usuarios` con usuarios distintos. Necesitamos autenticación que no colisione entre organizaciones.

Por defecto, Django usa `auth_user` con `id` autoincremental entero:
- Usuario id=5 en ORG1 es una persona distinta de usuario id=5 en ORG2
- Un JWT que dice "yo soy el 5" presentado en otro subdominio autenticaría como la persona equivocada

Esto es un agujero de seguridad.

**Alternativas Evaluadas**

1. **Entero autoincremental + validación en JWT** (descartada)
   - Requiere que cada request valide claim `tenant` + `user_id`
   - Un bug en ese middleware = autenticación cruzada
   - No es garantía estructural, es esperanza

2. **Entero autoincremental + prefijo global** (descartada)
   - Ej: ORG1 usuarios 1-999, ORG2 usuarios 1000-1999
   - Imposible de predecir, escala mal
   - Sigue siendo integer dentro de cada esquema

3. **UUID como PK (ELEGIDA ✓)**
   - `123e4567-e89b-12d3-a456-426614174000` es único en el planeta
   - No hay colisión posible entre organizaciones
   - Escalable: agregar orgs no requiere renumerar nada
   - JWT "yo soy 123e..." es válido globalmente

**Decisión**

Crear `AUTH_USER_MODEL = "usuarios.Usuario"` con:
- `id = UUIDField(primary_key=True, default=uuid4)`
- Antes del primer `python manage.py migrate`
- No usar `auth.User` de Django

Además: JWT siempre lleva claim `tenant` validado contra `Host`.

**Consecuencias**

*Positivas:*
- Garantía estructural: id=5 en ORG1 ≠ id=5 en ORG2
- Escalable: nuevas orgs no requieren renumerar usuarios
- Exportable: usuario UUID se puede copiar entre sistemas sin colisión
- URLs seguras: `/api/usuarios/123e4567.../` no filtra información

*Negativas:*
- UUID en URLs es menos legible que id=123
- URLs más largas en links (pero es trivial)
- Serialización JSON es string, no int
- Requisito de Python: `uuid` stdlib (incluido)

**Implicaciones Arquitectónicas**

1. Field declaración correcta:
   ```python
   from uuid import uuid4
   from django.db import models
   
   class Usuario(AbstractUser):
       id = models.UUIDField(primary_key=True, default=uuid4, editable=False)
       email = models.EmailField(unique=True)
       # No usar username
   
   class Meta:
       db_table = 'usuarios'
   ```

2. JWT claims:
   ```python
   {
       "sub": "123e4567-e89b-12d3-a456-426614174000",  # user uuid
       "tenant": "sinosofica_org1",                      # schema name
       "exp": 1234567890,
       "iat": 1234567000
   }
   ```

3. Validación en middleware (después de `TenantMainMiddleware`):
   ```python
   def validate_tenant_claim(request):
       if not hasattr(request, 'user') or not request.user.is_authenticated:
           return
       
       tenant_claim = request.auth.get('tenant') if request.auth else None
       schema_name = request.tenant.schema_name
       
       if tenant_claim != schema_name:
           raise AuthenticationFailed("Tenant mismatch")
   ```

4. Serializers: UUID se serializa como string:
   ```python
   class UsuarioSerializer(serializers.ModelSerializer):
       id = serializers.UUIDField(read_only=True)
       
       class Meta:
           model = Usuario
           fields = ['id', 'email', 'nombre', ...]
   ```

5. URLs: Usar `UUIDConverter` en Django:
   ```python
   # urls.py
   from django.urls import path, converters
   
   urlpatterns = [
       path('usuarios/<uuid:usuario_id>/', views.usuario_detail),
   ]
   ```

**Blindajes**

- Tests de autenticación cruzada:
  ```python
  def test_token_cross_org_rejected():
      org1_token = create_token(user=org1_user)
      client = Client(HTTP_HOST='org2.tudominio.mx')
      response = client.get('/api/usuarios/', HTTP_AUTHORIZATION=f'Bearer {org1_token}')
      assert response.status_code == 401
  ```

- Búsqueda de usuario por UUID + tenant:
  ```python
  # Nunca por id simple
  usuario = Usuario.objects.get(id=uuid, tenant=request.tenant)
  ```

- Swagger/OpenAPI: UUID documentado como string en schema

**Referencias**

- [Django AbstractUser](https://docs.djangoproject.com/en/5.1/topics/auth/customizing/#substituting-a-custom-user-model)
- [Python uuid module](https://docs.python.org/3/library/uuid.html)
- [UUID in PostgreSQL](https://www.postgresql.org/docs/16/datatype-uuid.html)
- ADR 001: Multi-tenancy por esquema
- CLAUDE.md: Invariantes de arquitectura

**Cambios en Procedimiento de Desarrollo**

- Antes del primer `migrate`: configurar `AUTH_USER_MODEL`
- Crear modelo `Usuario` en `apps/usuarios/models.py`
- Incluir en `INSTALLED_APPS` antes que `django.contrib.auth`
- Primera migración crea tabla `usuarios` con UUID PK

---

*Aprobado por: Alonso, arquitecto del sistema*
*Fecha: Septiembre 2026*
