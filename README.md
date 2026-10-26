# 🌙 FIAP Bank — O Turno da Noite

**Simulação de incidentes críticos em um sistema bancário na véspera da Black Friday**  
**Entrega:** registro das decisões das rodadas e relatório pós-mortem (3ª aula)

## 👥 Integrantes

- Samuel da Silva Nunes — RM 564435
- Miguel Vanucci Delgado — RM 563491
- João Vitor Lima Caldeira — RM 566541
- Henry dos Santos Lima — RM 565309

## 📌 Sobre o projeto

Este README documenta o percurso do grupo durante seis rodadas e um evento relâmpago da simulação **O Turno da Noite**. As situações envolveram transferências financeiras, consultas ao banco, geração de números de conta, integridade de extratos, criação de objetos e testes. Para cada decisão, registramos o **tipo**, o **voto**, a **justificativa com mecanismo técnico e trade-off** e o **placar acumulado**, conforme o modelo pedido pelo professor.

**Objetivos:** preservar a integridade financeira; prevenir SQL Injection; garantir identificadores únicos; manter os comprovantes consistentes; tratar falhas próximo à origem; e respeitar testes e contratos de negócio.

> **Legenda do placar:** 🔥, 💰 e 🧹 são os indicadores **da simulação**. Valores de 💰 são pontos do jogo, não valores em reais. O placar abaixo é acumulado, começando em 0 · 0 · 0. As consequências descritas correspondem ao enunciado da atividade, não a um deploy real verificado.

## 📊 Resumo das decisões e placar

| Etapa | Horário | Voto | Variação 🔥 | Variação 💰 | Variação 🧹 | Placar acumulado (🔥 / 💰 / 🧹) |
|---|---|:---:|---:|---:|---:|---|
| Rodada 1 — Saldo negativo | 19h12 | C | +1 | +35 | −1 | **1 / 35 / −1** |
| Rodada 2 — SQL Injection | 21h02 | D | 0 | 0 | −1 | **1 / 35 / −2** |
| Rodada 3 — Contas duplicadas | 22h50 | D | 0 | +10 | 0 | **1 / 45 / −2** |
| Rodada 4 — Extratos invertidos | 00h47 | B | +1 | +25 | −1 | **2 / 70 / −3** |
| Rodada 5 — Factory com `null` | 01h30 | A | −1 | 0 | 0 | **1 / 70 / −3** |
| Relâmpago C — Builder | 02h20 | B | 0 | 0 | 0 | **1 / 70 / −3** |
| Rodada 6 — Teste vermelho | 03h45 | B | −1 | +30 | −1 | **0 / 100 / −4** |

**Resultado final:** 🔥 **0** · 💰 **100** · 🧹 **−4**.

**Eventos condicionais não aplicáveis:** Relâmpago A (exigia R1 = D), Relâmpago B (exigia R2 = A) e Relâmpago D (exigia R2 = A ou C). O grupo participou do **Relâmpago C** porque escolheu **A na R5**.

---

## 💸 Rodada 1 — O saldo que virou negativo (19h12)

**Tipo:** rodada · **Voto:** C

**Justificativa:** `TransferenciaService.realizar()` lançava `SaldoInsuficienteException`, mas o `catch` ignorava a exceção e permitia que `origem.debitar(valor + taxa)` fosse executado. Escolhemos remover esse tratamento e escrever um teste com `assertThrows` antes da liberação. **Trade-off:** a correção com teste consumiu **40 minutos adicionais**, e **oito clientes foram afetados no total**, porém a suíte passou a proteger a regra de não transferir sem saldo.

**Placar do grupo após esta decisão:** 🔥 **1** · 💰 **35** · 🧹 **−1**

### Evidência e causa técnica

Três clientes apresentaram saldo negativo; um registrou **−R$ 8.940,00**. A API retornava sucesso para uma transferência que deveria ter sido recusada.

```java
try {
    if (origem.getSaldo() < valor + taxa) {
        throw new SaldoInsuficienteException("Saldo insuficiente");
    }
} catch (SaldoInsuficienteException e) {
    // Exceção ignorada
}
origem.debitar(valor + taxa);
```

### Correção proposta

```java
if (origem.getSaldo() < valor + taxa) {
    throw new SaldoInsuficienteException("Saldo insuficiente");
}
origem.debitar(valor + taxa);
destino.creditar(valor);
```

```java
@Test
void deveRecusarTransferenciaSemSaldo() {
    assertThrows(SaldoInsuficienteException.class,
        () -> service.realizar(origem, destino, 100.0, 5.0));
}
```

O teste também deve conferir que **ambos os saldos permanecem inalterados** após uma tentativa recusada. Investigar e reconciliar as contas afetadas, incluindo eventuais operações concluídas indevidamente.

**Aprendizado:** não se deve capturar e ignorar uma exceção que protege uma regra financeira.

---

## 🔐 Rodada 2 — A injeção que derreteu o banco (21h02)

**Tipo:** rodada · **Voto:** D

**Justificativa:** a busca por titular montava SQL por concatenação da entrada do usuário, permitindo alterar a lógica da consulta. Escolhemos utilizar `ContaRepository.findByTitular`, uma **derived query** já disponível no Spring Data JPA, para tratar o nome como valor parametrizado. **Trade-off:** cerca de **50 minutos de correção** (mais que os ~40 minutos da opção com `PreparedStatement`), em troca de aproveitar a arquitetura existente e **reduzir a dívida técnica**.

**Placar do grupo após esta decisão:** 🔥 **1** · 💰 **35** · 🧹 **−2**

### Evidência e causa técnica

O monitoramento apontou **latência p95 de 8,4 segundos** e crescimento nas requisições. Uma entrada como `OR '1'='1` permitiu o retorno de todas as contas.

```java
String sql = "SELECT * FROM contas WHERE titular = '" + nome + "'";
Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(sql);
```

### Correção proposta

```java
public interface ContaRepository extends JpaRepository<Conta, Long> {
    List<Conta> findByTitular(String titular);
}
```

```java
public List<Conta> buscarPorTitular(String titular) {
    return contaRepository.findByTitular(titular);
}
```

Testar entradas normais e maliciosas, verificar possíveis acessos indevidos em logs e acompanhar a latência após a alteração.

**Aprendizado:** parametrização protege a estrutura da consulta; bloquear caracteres suspeitos manualmente não é proteção equivalente.

---

## 🔢 Rodada 3 — A noite das contas gêmeas (22h50)

**Tipo:** rodada · **Voto:** D

**Justificativa:** o método `getInstancia()` criava um novo gerador em toda chamada, reiniciando a numeração em 1001 e produzindo duplicidades. Decidimos delegar a numeração ao banco por uma **sequence**, evitando depender do estado de um Singleton na aplicação. **Trade-off:** a migração exigiu **aproximadamente uma hora** e **+10 pontos de impacto financeiro**, mas estabeleceu uma estratégia centralizada, mais robusta para concorrência e reinicializações.

**Placar do grupo após esta decisão:** 🔥 **1** · 💰 **45** · 🧹 **−2**

### Evidência e causa técnica

Clientes diferentes receberam o número **1001**.

```java
public static GeradorNumeroConta getInstancia() {
    return new GeradorNumeroConta(); // sempre uma nova instância
}
```

### Correção proposta

Exemplo em PostgreSQL (a sequência inicial exige auditoria dos números já cadastrados):

```sql
CREATE SEQUENCE conta_numero_seq START WITH 1002 INCREMENT BY 1;
SELECT nextval('conta_numero_seq');
```

Também recomendamos, **após tratar as duplicatas existentes**, uma restrição de unicidade:

```sql
ALTER TABLE contas ADD CONSTRAINT uk_contas_numero UNIQUE (numero);
```

A sequence reduz o risco de novos números repetidos; **não corrige sozinha as duplicidades e extratos já existentes**. É preciso auditar as contas afetadas e testar aberturas simultâneas.

**Aprendizado:** um Singleton dentro da aplicação não é garantia suficiente de unicidade em ambientes distribuídos.

---

## 🧾 Rodada 4 — O comprovante que aponta para a pessoa errada (00h47)

**Tipo:** rodada · **Voto:** B

**Justificativa:** o serviço movimentava os saldos corretamente, mas registrava origem e destino invertidos no histórico. Escolhemos corrigir a ordem dos parâmetros e **reprocessar os extratos anteriores**, sem movimentar novamente o dinheiro. **Trade-off:** aproximadamente **duas horas** e **+25 pontos de impacto financeiro**, porém a solução também corrige a inconsistência histórica e reduz a dívida técnica.

**Placar do grupo após esta decisão:** 🔥 **2** · 💰 **70** · 🧹 **−3**

### Evidência e causa técnica

Ana (conta 1001) transferiu para Bob (conta 1002), mas o comprovante indicava o caminho inverso.

```java
Transferencia extrato = new Transferencia(
    tipo,
    destino.getNumero(), // incorreto: deveria ser origem
    origem.getNumero(),  // incorreto: deveria ser destino
    valor,
    taxa
);
```

### Correção proposta

```java
Transferencia extrato = new Transferencia(
    tipo,
    origem.getNumero(),
    destino.getNumero(),
    valor,
    taxa
);
```

Identificar registros afetados, corrigir **somente as informações do extrato**, verificar saldos e trilha de auditoria e disponibilizar comprovantes consistentes.

**Aprendizado:** integridade financeira envolve também rastreabilidade e confiabilidade dos registros históricos.

---

## 🏭 Rodada 5 — A fábrica que devolve null (01h30)

**Tipo:** rodada · **Voto:** A

**Justificativa:** a `ContaFactory` retornava `null` para tipos não reconhecidos, e o erro só aparecia no `save()` do repositório, distante da causa real. Escolhemos **fail-fast**, lançando uma exceção explícita na própria Factory. **Trade-off:** o erro fica claro em cerca de **30 minutos**, sem penalidade no indicador de dívida técnica, **mas a opção A não implementa a conta `UNIVERSITARIA`**: o cadastro desse produto continua indisponível enquanto o tipo não for adicionado corretamente.

**Placar do grupo após esta decisão:** 🔥 **1** · 💰 **70** · 🧹 **−3**

### Evidência e causa técnica

O cadastro `POST /api/contas` com tipo `UNIVERSITARIA` causava `Entity must not be null`.

```java
return switch (tipo) {
    case "CORRENTE" -> new Corrente(...);
    case "POUPANCA" -> new Poupanca(...);
    default -> null;
};
```

### Correção proposta

```java
return switch (tipo) {
    case "CORRENTE" -> new Corrente(...);
    case "POUPANCA" -> new Poupanca(...);
    default -> throw new IllegalArgumentException(
        "Tipo de conta inválido: " + tipo);
};
```

**Ação futura importante:** implementar e testar o caso `UNIVERSITARIA`, já que a campanha anunciou esse produto como disponível. A API deve retornar uma resposta apropriada ao cliente, em vez de expor detalhes internos.

**Aprendizado:** identificar um erro na origem facilita diagnóstico, mas não substitui a implementação da funcionalidade faltante.

---

## ⚡ Evento relâmpago C — O cadastro com metade dos campos (02h20)

**Tipo:** relâmpago · **Voto:** B

**Justificativa:** o marketing pediu cadastro com menos campos, enquanto o construtor atual exigia vários argumentos. Escolhemos o padrão **Builder**, que permite configurar campos gradualmente e concentrar valores padrão e validações no `build()`. **Trade-off:** acrescenta uma camada de construção, mas evita a proliferação de construtores e o uso indiscriminado de `null`; a opção não altera a dívida técnica no placar.

**Placar do grupo após esta decisão:** 🔥 **1** · 💰 **70** · 🧹 **−3**

### Exemplo conceitual

```java
Conta conta = Conta.builder()
    .nome(nome)
    .saldo(saldo)
    .build();
```

Os valores padrão devem ser usados **somente onde as regras permitirem**. Campos obrigatórios, como identificações exigidas para uma conta bancária válida, não devem ser omitidos sem aprovação da regra de negócio.

**Aprendizado:** flexibilidade na construção não significa aceitar objetos inválidos.

---

## 🧪 Rodada 6 — O teste vermelho às 03h45 (janela fecha às 04h)

**Tipo:** rodada · **Voto:** B

**Justificativa:** o teste de TED esperava **R$ 5,00**, conforme contrato, mas a implementação cobrava **R$ 50,00**. Escolhemos **investigar a falha e não mascará-la** com `@Disabled`, alterando a expectativa do teste ou liberando a suíte vermelha. **Trade-off:** **perdemos a janela das 04h00**, de modo que as correções da noite só poderiam entrar **depois da abertura da campanha**, e recebemos **+30 pontos de impacto financeiro**; por outro lado, preservamos um teste válido e identificamos a divergência contratual real.

**Placar do grupo após esta decisão:** 🔥 **0** · 💰 **100** · 🧹 **−4**

### Evidência e causa técnica

```text
Tests run: 13, Failures: 1
TransferenciaServiceTest.tedDeveCobrarTaxaDe5Reais:
expected: <5.0> but was: <50.0>
```

```java
@Test
void tedDeveCobrarTaxaDe5Reais() {
    assertEquals(5.0, service.calcularTaxa("TED"), 0.0001);
}
```

### Correção proposta

```java
private static final double TAXA_TED = 5.00;
```

Confirmar a regra contratual, corrigir o valor no código e executar toda a suíte antes de uma nova liberação. Em sistemas monetários reais, utilizar `BigDecimal` para cálculos de dinheiro.

**Aprendizado:** um teste vermelho que revela quebra de contrato não deve ser desabilitado ou alterado apenas para liberar a aplicação.

---

# 🌅 3ª aula — Relatório pós-mortem

## 1. Linha do tempo da madrugada

| Horário | Fato e consequência da escolha do grupo |
|---|---|
| **19h12** | Saldo negativo: opção **C** corrigiu o fluxo e acrescentou teste, mas o tempo adicional levou a **8 clientes afetados no total**. |
| **20h10** | Relâmpago A **não aplicável**: só ocorreria se tivéssemos escolhido rollback (D) na R1. |
| **21h02** | SQL Injection: opção **D** substituiu a consulta vulnerável por derived query, com ~50 min de implementação. |
| **22h40** | Relâmpago B **não aplicável**: dependia da escolha A na R2. |
| **22h50** | Contas duplicadas: opção **D** escolheu uma sequence no banco, com ~1 h de migração. |
| **00h47** | Extratos invertidos: opção **B** corrigiu novos registros e o histórico, com ~2 h de reprocessamento. |
| **01h30** | Factory retornando `null`: opção **A** adotou fail-fast, mas ainda faltava implementar o produto universitário. |
| **02h20** | Relâmpago C: opção **B** escolheu Builder para organizar parâmetros opcionais e validações. |
| **03h20** | Relâmpago D **não aplicável**: dependia de A ou C na R2. |
| **03h45** | Teste TED vermelho: opção **B** preservou o teste e investigou a taxa incorreta. |
| **04h00** | **Janela perdida**: as correções planejadas ficam para depois da abertura da campanha. |

## 2. O caminho do grupo: escolhas, causas e consequências

A equipe adotou, em geral, decisões orientadas à **correção da causa técnica**, mesmo quando isso exigia mais tempo. Na primeira rodada, preferimos um teste de regressão a um hotfix sem cobertura. Na segunda, escolhemos a abstração existente do repositório, eliminando o SQL construído manualmente. Na terceira, preferimos uma numeração centralizada no banco em vez de confiar apenas em um Singleton.

Na quarta rodada, optamos por reparar também o histórico dos comprovantes, não apenas os novos registros. Na quinta, fizemos o erro aparecer na Factory por meio de fail-fast; reconhecemos que **isso não entregava o produto universitário**, que ainda necessitava de implementação. No relâmpago C, escolhemos Builder com defaults controlados. Por fim, **na rodada 6 decidimos investigar o teste vermelho**, aceitando perder a janela de deploy em vez de permitir que uma taxa em desacordo com o contrato fosse tratada como correta.

**Efeito agregado do caminho:** 🔥 **0**, 💰 **100**, 🧹 **−4**. O saldo negativo de dívida técnica é favorável dentro das regras da simulação, mas o placar não elimina o atraso de lançamento nem as pendências técnicas identificadas.

## 3. Causa raiz de dois incidentes

### Incidente A — Saldo negativo (rodada 1)

**Aula/conceitos:** exceções em Java, fluxo de controle, regras de negócio e testes com JUnit.

- **Causa raiz:** um `catch (SaldoInsuficienteException)` suprimia a exceção, e o programa prosseguia para `origem.debitar(valor + taxa)`.
- **Mecanismo:** a violação de saldo era detectada, mas não interrompia a operação. O débito seguia mesmo quando a transferência deveria falhar.
- **Impacto na simulação:** oito clientes afetados no total após a decisão C; um dos saldos observados era de **−R$ 8.940,00**.
- **Prevenção:** propagar a exceção, testar `assertThrows`, verificar saldos inalterados e rodar regressão antes de qualquer deploy.

### Incidente B — SQL Injection (rodada 2)

**Aula/conceitos:** JDBC, composição de SQL, parametrização e Spring Data JPA.

- **Causa raiz:** a entrada do usuário era concatenada diretamente em `SELECT * FROM contas WHERE titular = '...'`.
- **Mecanismo:** uma entrada como `' OR '1'='1` podia alterar o predicado SQL e fazer a consulta retornar registros além do titular solicitado.
- **Impacto na simulação:** o banco chegou a retornar todas as contas e o monitoramento indicou p95 de 8,4 s.
- **Prevenção:** utilizar `findByTitular`, testar entradas hostis e auditar os logs para avaliar possível acesso indevido.

## 4. O que faríamos diferente em duas rodadas

### Rodada 1 — Conter a exposição enquanto corrigimos

**Nossa escolha original:** C, correção com teste.

**Mudança proposta:** manter o desenvolvimento com teste, mas introduzir **contenção temporária e direcionada** no fluxo vulnerável, com comunicação operacional aos responsáveis, enquanto a alteração definitiva é validada. Priorizaríamos a reconciliação das contas afetadas.

**Motivo:** os 40 minutos adicionais resultaram em novos clientes afetados. A contenção poderia reduzir esse risco; por outro lado, bloquear transferências tem custo operacional e precisa ser usado apenas enquanto necessário.

### Rodada 6 — Antecipar os testes de contrato

**Nossa escolha original:** B, investigar e perder a janela.

**Mudança proposta:** executar a suíte completa e os testes de regras financeiras durante toda a madrugada, incluindo uma checagem antecipada da taxa TED de R$ 5,00, em vez de descobrir o defeito às 03h45.

**Motivo:** detectar o teste vermelho mais cedo daria margem para investigar e corrigir com segurança antes das 04h00. Se ainda restasse uma divergência contratual, manteríamos a decisão de **não liberar** uma versão incorreta.

## 5. Apresentação relâmpago — “A madrugada do nosso grupo” (2 a 3 minutos)

**Samuel — Introdução, R1 e R2 (~40 s)**  
“Nosso grupo assumiu o plantão do FIAP Bank na véspera da Black Friday. O primeiro incidente permitia transferências sem saldo, porque uma exceção era ignorada. Escolhemos corrigir e escrever um teste, apesar do tempo adicional e dos oito clientes afetados. Depois, encontramos uma SQL Injection na busca por titular. Preferimos a derived query do Spring Data JPA, pois ela aproveitava a arquitetura existente e evitava montar SQL com a entrada do usuário.”

**Miguel — R3 e R4 (~35 s)**  
“Na terceira rodada, duas contas tinham o mesmo número porque o suposto Singleton criava um novo gerador a cada chamada. Escolhemos transferir a numeração para uma sequence no banco. Mais tarde, os extratos exibiam remetente e destinatário invertidos, embora os saldos estivessem certos. Decidimos corrigir a gravação e reprocessar os registros antigos para recuperar a confiabilidade do histórico.”

**João — R5 e relâmpago C (~35 s)**  
“Na quinta rodada, a Factory devolvia `null` para a conta universitária, e o erro aparecia só ao salvar. Usamos fail-fast para mostrar a causa imediatamente, mas entendemos que ainda seria necessário implementar esse tipo de conta. Em seguida, o marketing pediu um cadastro com menos campos. Escolhemos Builder, concentrando valores padrão e validações no momento de criar o objeto.”

**Henry — R6, placar e conclusão (~40 s)**  
“Às 03h45, descobrimos que a TED cobrava cinquenta reais, embora o contrato exigisse cinco. Escolhemos investigar, sem desabilitar o teste nem alterar a expectativa, e acabamos perdendo a janela das quatro horas. Nosso placar terminou em zero de fogo, cem pontos de impacto financeiro e menos quatro de dívida técnica. Aprendemos que testes, segurança e integridade de dados precisam valer mais do que um deploy apressado.”

## 6. Go/No-Go e conclusão

**Resultado na simulação: NO-GO na janela das 04h00.** A escolha B da rodada 6 implicou perder a janela; as correções da noite só poderiam entrar depois da abertura da campanha. Isso **não comprova que um deploy posterior foi realizado**, nem que a conta universitária já foi implementada.

Nossas escolhas privilegiaram integridade de saldos, segurança das consultas, numeração consistente, rastreabilidade dos comprovantes e testes confiáveis. O custo foi tempo e impacto financeiro no jogo. Para um próximo plantão, adotaríamos contenção de incidentes mais cedo, verificação antecipada das regras financeiras e uma lista clara de pendências antes de liberar novas versões.

## 7. Próximas ações técnicas

- Auditar e reconciliar transferências que geraram saldos negativos.
- Investigar possível exposição de dados no episódio de SQL Injection.
- Auditar contas duplicadas antes de implantar `UNIQUE` e inicializar a sequence com segurança.
- Validar a reconstrução dos extratos sem repetir movimentações financeiras.
- Implementar e testar explicitamente o tipo `UNIVERSITARIA` na Factory.
- Verificar quais campos do cadastro podem receber defaults sem violar regras do produto.
- Corrigir a taxa TED conforme o contrato e executar testes unitários e de regressão (`./mvnw clean test`) antes da próxima liberação.

---
