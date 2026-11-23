  # Evolução da Solução em Escala 

Para aplicar essa validação em uma empresa com várias contas AWS e diferentes squads, eu evoluiria a solução em quatro frentes práticas:

**1. Isolamento e bloqueios base (AWS Organizations e SCPs)**
Separaria os ambientes (Dev, Homolog e Prod) e as squads em contas distintas dentro do AWS Organizations para reduzir o impacto de eventuais falhas. Nas Unidades Organizacionais (OUs), aplicaria Service Control Policies (SCPs) como barreira inicial para impedir ações críticas ou fazer alterações manuais via Console fora das esteiras de deploy.

**2. Políticas centralizadas e validação nas esteiras (CI/CD)**
As políticas ficariam em um repositório central gerenciado pelo time de segurança. As squads manteriam autonomia para fazer seus deploys, mas os workflows padronizados das esteiras (GitHub Actions) consumiriam essas políticas centrais e rodariam o `c7n-left` antes de subir a infraestrutura, bloqueando o deploy caso alguma regra fosse violada.

**3. Monitoramento contínu e legado**
Como a esteira valida apenas o que passa pelo código, usaria o AWS Config junto com o Cloud Custodian em modo runtime (via Lambda) para vigiar alterações feitas diretamente na nuvem  Para o ambiente legado já existente, começaria rodando as regras apenas em modo auditoria para mapear os riscos sem quebrar sistemas em produção, integrando os alertas com ferramentas como o Jira para abrir chamados e tratar as correções em ondas junto aos times donos dos recursos.

**4. Limitações e cuidados na operação**
* **SCPs têm escopo limitado:** Elas bloqueiam permissões de API, mas não validam configurações detalhadas de um recurso; por isso precisam trabalhar em conjunto com o `c7n-left` e o AWS Config.
* **Risco de travar a operação:** Bloqueios rígidos exigem um processo claro de exceção temporária e acessos emergenciais (*break-glass*) auditados para não atrasar incidentes críticos.
* **Custo e manutenção:** Manter o AWS Config ativo em muitas contas gera custo recorrente e exige revisão constante das regras para não gerar excesso de alertas (falsos positivos) nas filas das equipes.