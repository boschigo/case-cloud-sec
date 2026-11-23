# Validação de Segurança em IaC — AWS S3 (Cloud Custodian)

Projeto de análise de riscos e automação preventiva de controles de segurança em arquivos Terraform usando Cloud Custodian (`c7n-left`), sem necessidade de credenciais ou acesso à nuvem.

## Dependências

* **Python 3.9+**
* **Cloud Custodian Shift-Left (`c7n-left`)**:
  ```bash
  pip install c7n-left
  ```
* **Terraform CLI** *(opcional, apenas para checagem local de formatação/sintaxe)*

---

## Instruções de Execução

Todos os comandos abaixo devem ser executados a partir da raiz do repositório (`case_cloud_sec/`) e funcionam 100% offline:

1. **Validar o cenário não conforme (deve falhar e retornar alerta de violação):**
   ```bash
   c7n-left run -p policies/policies_s3.yml -d testes/bucket_inseguro
   ```

2. **Validar o cenário conforme (deve passar sem alertas):**
   ```bash
   c7n-left run -p policies/policies_s3.yml -d testes/bucket_seguro
   ```

---

## Decisões Técnicas

* **Provedor único (AWS):** A análise manual (`tf/`) e as regras automatizadas foram padronizadas exclusivamente para recursos AWS, mantendo consistência entre o diagnóstico, as políticas e os testes.
* **Escolha do `c7n-left`:** Optei pelo modo shift-left do Cloud Custodian porque ele lê os arquivos `.tf` estaticamente, permite escrever regras em YAML legível e roda direto no terminal ou em pipelines CI/CD sem exigir chaves de acesso da AWS.
* **Priorização do S3 na automação:** Como recorte inicial, automatizei as regras de exposição pública de buckets S3 pelo impacto direto em vazamento de dados. Os demais recursos analisados (`rds.tf`, `security-group.tf` e `ebs.tf`) estão documentados na análise de riscos (`analise.xlsx`).
* **Estrutura separada:** Mantive os arquivos originais em `tf/` isolados dos cenários de teste em `testes/` (`bucket_inseguro` e `bucket_seguro`) para facilitar a reprodução dos testes de `FAIL` e `PASS`. A integração com esteira está demonstrada em `.github/workflows/security-check.yml`.

---

## Controles Implementados

Arquivo: `policies/policies_s3.yml`

* **`s3-bloqueio-publico-incompleto`** (`severity: high`)
  * Avalia o recurso `terraform.aws_s3_bucket_public_access_block`.
  * Reprova o código caso qualquer uma das 4 flags (`block_public_acls`, `block_public_policy`, `ignore_public_acls`, `restrict_public_buckets`) esteja como `false`.

* **`s3-politica-permissiva`** (`severity: critical`)
  * Avalia o recurso `terraform.aws_s3_bucket_policy`.
  * Usa expressão regular para bloquear políticas que combinem `"Effect": "Allow"` com `"Principal": "*"` (acesso anônimo público).
  * O filtro foi escrito exigindo `Effect: Allow` junto com `Principal: *` para não gerar falso positivo em regras legítimas de bloqueio (`Effect: Deny` com `Principal: *`, como políticas que exigem HTTPS).

---

## Limitações

* **Cobertura parcial de recursos:** Apenas os controles de S3 foram automatizados nesta entrega. Os problemas identificados em `rds.tf`, `security-group.tf` e `ebs.tf` constam apenas no mapeamento manual e ainda não possuem políticas YAML associadas.
* **Análise estática limitada:** O `c7n-left` valida a declaração do código HCL; configurações herdadas de módulos externos remotos ou alterações feitas manualmente no console AWS não são detectadas aqui.
* **Próximos passos:** Criar políticas equivalentes para criptografia em RDS/EBS e restrição de portas abertas em Security Groups, além de centralizar o arquivo de políticas para reutilização em múltiplos repositórios (detalhado em `evolucao.md`).