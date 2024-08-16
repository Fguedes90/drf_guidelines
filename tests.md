# 01 - Diretrizes para Testes

## Diretrizes para Testes em APIs Django com factory_boy e APITestCase`factory_boy` e `APITestCase`

### 1. Introdução

O objetivo deste guia é estabelecer padrões claros e consistentes para a nomenclatura de testes em APIs desenvolvidas com Django, utilizando `factory_boy` e `APITestCase`. O uso de nomenclaturas significativas e padronizadas facilita a compreensão e manutenção do código de teste.

### 2. Diretrizes de Nomeação

### 2.1 Seja Descritivo e Claro

- Os nomes das funções de teste devem descrever claramente o que está sendo testado e qual o comportamento esperado.

### 2.2 Padrão de Nomenclatura

- Todas as funções de teste devem começar com “test_” para que possam ser detectadas pelo pytest.
- Utilize padrões de nomenclatura como:
    - **`test_should` + [ação] + [condição]**: Exemplo: `test_should_return_200_when_user_is_authenticated`.
    - **`test_` + [ação] + `When` + [condição]**: Exemplo: `test_return_200_when_user_is_authenticated`.

### 2.3 Separação de Palavras

- Use underscores (`_`) para separar palavras, melhorando a legibilidade. Exemplo: `test_should_return_404_when_resource_not_found`.

### 2.4 Cenários e Resultados

- Inclua o cenário específico e o resultado esperado no nome da função. Exemplo: `test_create_user_should_return_201_when_data_is_valid`.

### 3. Exemplos Práticos

### Testando uma API de Autenticação

1. Bom: `test_should_return_200_when_user_is_authenticated`
    - Ruim: `should_return_200_when_user_is_authenticated`
2. Bom: `test_should_return_401_when_user_is_not_authenticated`
    - Ruim: `test_auth`

### Testando uma API de Recursos

1. Bom: `test_should_return_404_when_resource_not_found`
    - Ruim: `get_resource`
2. Bom: `test_create_user_should_return_201_when_data_is_valid`
    - Ruim: `create_user`

### 4. Exemplo de Implementação em Django

### 4.1 Estrutura do Diretório de Testes

As **factories** devem ser armazenadas em um arquivo chamado `factories.py` dentro do diretório `tests` do seu aplicativo. Isso ajuda a manter a organização e facilita a reutilização das **factories** em múltiplos arquivos de teste.

```
/myapp
    /tests
        __init__.py
        factories.py
        test_authentication.py
        test_resources.py
```

### 4.2 Exemplo de `factories.py`

```python
# tests/factories.pyfrom factory import DjangoModelFactory, Faker
from myapp.models import User, Resource
class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
    username = Faker('user_name')
    password = Faker('password')
class ResourceFactory(DjangoModelFactory):
    class Meta:
        model = Resource
    name = Faker('name')
    description = Faker('text')
```

### 4.3 Utilização do `FastTenantAPITestCase`

Ao testar models, serializers e views, deve-se utilizar a classe de teste personalizada `FastTenantAPITestCase`. Essa classe é projetada para facilitar o teste em contextos de multitenancy.

### 4.4 Acesso ao Modelo de Usuário

Quando for necessário acessar o modelo de usuário, deve-se utilizar o método `get_user_model` do Django para garantir que a implementação correta do modelo de usuário seja utilizada.

### 5. Exemplo de Teste com `FastTenantAPITestCase`

```python
# tests/test_models.pyfrom django.contrib.auth import get_user_model
from api.common.fast_tenant_api_test_case import FastTenantAPITestCase
from api.tenant.models import MemberRoles
from api.tenant.tests.factories import RoleFactory, UserFactory
User = get_user_model()
class OrganizationMemberModelTest(FastTenantAPITestCase):
    def setUp(self) -> None:
        super().setUp()
        self.user_owner = UserFactory(is_staff=True)
        self.normal_user = UserFactory()
        self.roleOne = RoleFactory(name="Role One")
        self.roleTwo = RoleFactory(name="Role Two")
    def test_should_add_member_when_valid_user_is_provided(self) -> None:
        member = MemberRoles.objects.create(user=self.normal_user)
        member.save()
        self.assertEqual(member.user, self.normal_user)
```

### 6. Dicas Adicionais

- **Configuração do Teste (`setUp`)**: Utilize o método `setUp` para preparar dados que seus testes necessitam.
- **Uso do `reverse`**: Utilize o método `reverse` do Django para gerar URLs a partir dos nomes das rotas. Isso facilita a manutenção do código.
- **Verificações Detalhadas**: Além dos códigos de status HTTP, verifique o conteúdo das respostas e outros detalhes importantes.

### 7. Estrutura de Camadas e Lógica de Abstração

### 7.1 Isolamento de Lógica nos Modelos

- A lógica de negócios e regras específicas devem ser implementadas nos métodos do modelo.
- Métodos de classe ou de instância são adequados para operações específicas do modelo.

```python
from django.db import models
class Product(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    discount = models.DecimalField(max_digits=10, decimal_places=2, default=0.0)
    def final_price(self):
        return self.price - self.discount
    def apply_discount(self, discount_amount):
        self.discount = discount_amount
        self.save()
```

### 7.2 Utilização de Serializers

- Os serializers devem ser utilizados para validações e transformações de dados.

```python
from rest_framework import serializers
from .models import Product
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'discount', 'final_price']
    final_price = serializers.DecimalField(max_digits=10, decimal_places=2, read_only=True)
    def validate_discount(self, value):
        if value < 0:
            raise serializers.ValidationError("Discount cannot be negative")
        return value
```

### 7.3 Simplificação das Views

- As views devem ser simples, delegando a lógica de validação para os modelos e serializers.

```python
from rest_framework import generics
from .models import Product
from .serializers import ProductSerializer
class ProductListCreateView(generics.ListCreateAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
class ProductDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    def perform_update(self, serializer):
        instance = serializer.save()
        instance.apply_discount(serializer.validated_data.get('discount', 0.0))
```

### 8. Conclusão

Seguir estas diretrizes ajudará a criar uma base de testes sólida, garantindo que a API funcione conforme o esperado e mantenha um código claro e manutenível. Boas práticas de testes não só melhoram a qualidade do código, mas também facilitam a colaboração em equipe e a evolução do projeto ao longo do tempo.