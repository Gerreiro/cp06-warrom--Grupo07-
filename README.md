🌙 FIAP Bank — O Turno da Noite
Simulação de incidentes críticos em um sistema bancário na véspera da Black Friday.
📌 Sobre o projeto

Integrantes - Samuel da Silva Nunes - RM 564435
              Miguel Vanucci Delgado - RM563491
              JOAO VITOR LIMA CALDEIRA - RM566541
              Henry dos Santos Lima - RM565309

Este documento registra três incidentes enfrentados pela equipe de desenvolvimento durante o plantão noturno do FIAP Bank.

Os problemas envolvem integridade financeira, segurança de consultas ao banco de dados e unicidade dos números de conta. O objetivo é identificar as causas, escolher soluções adequadas e reduzir o risco de recorrência.

🎯 Objetivos
Impedir transferências sem saldo suficiente.
Proteger consultas contra SQL Injection.
Garantir identificadores únicos para as contas.
Implementar testes automatizados para regras críticas.
Priorizar a integridade dos dados e a segurança dos clientes.
💸 Rodada 1 — O saldo que virou negativo

Horário: 19h12

Problema

Três clientes apresentaram saldo negativo após transferências. Uma das contas ficou com saldo de R$ 8.940,00 negativos.

A causa está no tratamento inadequado da exceção de saldo insuficiente:

try {
    if (origem.getSaldo() < valor + taxa) {
        throw new SaldoInsuficienteException(
            "Saldo insuficiente"
        );
    }
} catch (SaldoInsuficienteException e) {
    // Exceção ignorada
}

origem.debitar(valor + taxa);


A exceção é capturada e ignorada, permitindo que o débito seja executado mesmo quando o saldo não é suficiente.

Decisão

Opção C — Corrigir a regra e escrever o teste antes de liberar a alteração.

Correção proposta
if (origem.getSaldo() < valor + taxa) {
    throw new SaldoInsuficienteException(
        "Saldo insuficiente"
    );
}

origem.debitar(valor + taxa);
destino.creditar(valor);


O débito só deve ocorrer após a validação do saldo.

Teste automatizado
@Test
void deveRecusarTransferenciaSemSaldo() {
    assertThrows(
        SaldoInsuficienteException.class,
        () -> service.realizar(
            origem, destino, 100.0, 5.0
        )
    );
}


O teste também deve confirmar que os saldos de origem e destino permanecem inalterados após uma tentativa recusada.

Plano de ação
Conter temporariamente novas transferências indevidas, se necessário.
Remover o tratamento que ignora a exceção.
Testar saldo insuficiente, saldo exato e saldo suficiente.
Investigar as contas afetadas e reconciliar os valores.
Executar a suíte de regressão antes do deploy.
Aprendizado

Exceções que representam violações de regras financeiras não devem ser ignoradas. Uma operação recusada precisa falhar sem produzir alterações indevidas nos saldos.

🔐 Rodada 2 — A injeção que derreteu o banco

Horário: 21h02

Problema

O endpoint de busca por titular utiliza concatenação de strings para construir uma consulta SQL.

String sql =
    "SELECT * FROM contas WHERE titular = '" + nome + "'";

Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(sql);


Uma entrada maliciosa pode modificar a estrutura da consulta e retornar dados que não deveriam ser acessíveis.

O monitoramento também registrou latência p95 de 8,4 segundos e aumento no volume de requisições.

Decisão

Opção D — Utilizar a derived query findByTitular do Spring Data JPA.

O repositório já possui um método apropriado, evitando a necessidade de montar manualmente a consulta SQL.

Correção proposta
public interface ContaRepository
        extends JpaRepository<Conta, Long> {

    List<Conta> findByTitular(String titular);
}


No serviço:

public List<Conta> buscarPorTitular(String titular) {
    return contaRepository.findByTitular(titular);
}


O Spring Data JPA gera a consulta e trata o titular como um valor, em vez de permitir que a entrada seja incorporada diretamente à sintaxe SQL.

Plano de ação
Substituir a consulta vulnerável pela derived query.
Confirmar que o endpoint utiliza o método seguro.
Restringir temporariamente o endpoint se a correção não puder ser implantada imediatamente.
Criar testes com nomes comuns, aspas e entradas maliciosas.
Verificar logs e investigar possível exposição de dados.
Monitorar latência e volume de requisições após a correção.
Aprendizado

Consultas parametrizadas são essenciais para prevenir SQL Injection. A validação de entradas pode complementar a proteção, mas não substitui a parametrização.

🔢 Rodada 3 — A noite das contas gêmeas

Horário: 22h50

Problema

Clientes diferentes receberam o mesmo número de conta, 1001.

A causa está no método que deveria implementar o padrão Singleton:

public static GeradorNumeroConta getInstancia() {
    return new GeradorNumeroConta();
}


O método cria uma nova instância a cada chamada. Se o contador é reiniciado no construtor, diferentes instâncias podem gerar números repetidos.

Decisão

Opção D — Delegar a geração dos números ao banco de dados.

Uma sequence centralizada é mais robusta do que depender exclusivamente de um contador mantido na aplicação.

Exemplo com PostgreSQL
CREATE SEQUENCE conta_numero_seq
    START WITH 1002
    INCREMENT BY 1;


Para obter um novo número:

SELECT nextval('conta_numero_seq');


O valor inicial é ilustrativo. Antes da implantação, é necessário auditar os números existentes e definir um ponto de partida seguro.

Proteção adicional

A coluna que armazena o número da conta deve possuir uma restrição de unicidade, para que o próprio banco rejeite duplicidades.

Exemplo conceitual:

ALTER TABLE contas
ADD CONSTRAINT uk_contas_numero UNIQUE (numero);


A restrição só deve ser criada após identificar e resolver as duplicidades existentes.

Plano de ação
Conter temporariamente novas aberturas de conta se a unicidade não puder ser garantida.
Auditar os números duplicados.
Identificar os titulares e as operações vinculadas a cada registro.
Implementar uma estratégia centralizada de geração de números.
Adicionar uma restrição UNIQUE.
Testar a criação simultânea de contas e o comportamento após reinicializações.
Aprendizado

Corrigir o Singleton pode resolver o defeito imediato dentro de uma instância da aplicação, mas não garante unicidade entre vários servidores ou após reinicializações. A estratégia de identificação precisa considerar concorrência e persistência.

🧪 Estratégia de testes

A suíte de testes deve cobrir, no mínimo:

Área	Cenário
Transferências	Recusar operações sem saldo suficiente
Transferências	Preservar os saldos quando a operação falha
Segurança	Rejeitar tentativas de alterar a consulta por meio de SQL Injection
Consultas	Retornar somente os registros correspondentes ao titular
Numeração	Impedir números de conta duplicados
Concorrência	Criar contas simultaneamente sem repetir identificadores
🚀 Validação antes do deploy

Executar a suíte automatizada:

./mvnw clean test


Antes da liberação, revisar as alterações, verificar os testes relevantes e confirmar que os incidentes não introduziram novas falhas.

Se a correção não puder ser validada com segurança, a equipe deve conter o risco e escalar a decisão operacional, em vez de liberar uma versão sabidamente vulnerável.

📋 Resumo das decisões
Rodada	Incidente	Decisão
1	Saldo negativo	C — Corrigir e testar
2	SQL Injection	D — Usar derived query
3	Contas duplicadas	D — Delegar ao banco
📚 Conclusão

Os incidentes demonstram que um sistema bancário confiável depende de três pilares:

Integridade financeira: operações devem respeitar as regras de saldo e preservar os dados quando falham.
Segurança: entradas dos usuários não podem alterar a estrutura das consultas ao banco.
Consistência dos dados: identificadores devem ser únicos mesmo sob concorrência e reinicializações.

A prioridade do plantão é corrigir a causa raiz, proteger os clientes, preservar os registros para auditoria e validar cada alteração antes da implantação.

Projeto: FIAP Bank
Contexto: Simulação de plantão noturno
Tecnologias abordadas: Java, Spring Data JPA, JDBC, SQL e JUnit

FIAP Bank — Plantão de Incidentes
Simulação de incidentes técnicos durante a preparação para a Black Friday.
📌 Sobre o projeto

Este documento reúne as decisões técnicas tomadas durante o plantão de incidentes do FIAP Bank. O objetivo é corrigir falhas, preservar a integridade dos dados financeiros e garantir que as entregas respeitem os contratos com os clientes.

🎯 Objetivos
Corrigir inconsistências nos registros de transferências.
Melhorar o tratamento de erros na criação de contas.
Permitir a construção flexível de objetos sem comprometer sua validade.
Garantir que as taxas cobradas respeitem as regras de negócio.
Evitar deploys com falhas conhecidas em funcionalidades financeiras.
🔄 Rodada 4 — O comprovante que aponta para a pessoa errada

Horário: 00h47

Problema

O sistema realiza a transferência corretamente, mas registra os números de origem e destino invertidos no extrato.

Transferencia extrato = new Transferencia(
    tipo,
    destino.getNumero(),
    origem.getNumero(),
    valor,
    taxa
);


Os saldos estão corretos, porém o histórico apresenta informações incorretas sobre quem enviou e quem recebeu o dinheiro.

Decisão

Opção B — Corrigir a gravação e reprocessar os extratos antigos.

Solução
Corrigir a ordem dos números de origem e destino.
Identificar os registros históricos afetados.
Corrigir os extratos sem movimentar novamente o dinheiro.
Validar a integridade dos registros e dos saldos.
Comunicar os clientes afetados e disponibilizar comprovantes corrigidos.
Aprendizado

A integridade financeira não depende apenas dos saldos. Os registros históricos também precisam refletir corretamente as operações realizadas.

🏭 Rodada 5 — A fábrica que devolve null

Horário: 01h30

Problema

A campanha de Conta Universitária com cashback entra no ar, mas a criação da conta falha com o erro:

Entity must not be null


A causa está na factory, que não reconhece o tipo UNIVERSITARIA e retorna null.

default -> null;


O erro só aparece posteriormente, durante a persistência no repositório.

Decisão

Opção A — Fail-fast, com tratamento explícito dos tipos não suportados.

A decisão precisa incluir a implementação do tipo UNIVERSITARIA, já que ele é anunciado como um produto válido.

Solução
public Conta criar(String tipo) {
    return switch (tipo) {
        case "CORRENTE" -> new Corrente(...);
        case "POUPANCA" -> new Poupanca(...);
        case "UNIVERSITARIA" -> new Universitaria(...);
        default -> throw new IllegalArgumentException(
            "Tipo de conta inválido ou não suportado: " + tipo
        );
    };
}

Ações complementares
Validar o tipo de conta recebido pela API.
Retornar mensagens de erro claras e apropriadas.
Criar testes para todos os tipos suportados.
Impedir que uma conta desconhecida seja substituída silenciosamente por outro tipo.
Aprendizado

Uma factory não deve retornar null silenciosamente quando não consegue criar um objeto. Erros devem ser detectados próximos à sua origem.

⚡ Evento Relâmpago C — O cadastro com metade dos campos

Horário: 02h20

Problema

O marketing deseja permitir cadastros informando apenas nome e saldo, enquanto o construtor atual exige todos os campos.

Decisão

Opção B — Builder com valores padrão.

Exemplo
Conta conta = Conta.builder()
    .nome(nome)
    .saldo(saldo)
    .build();

Diretrizes de implementação
Permitir a configuração gradual dos campos.
Definir valores padrão somente para campos realmente opcionais.
Validar os campos obrigatórios durante a construção.
Confirmar as regras de negócio para CPF e tipo de conta.
Não permitir que campos ausentes gerem objetos inválidos.
Aprendizado

O padrão Builder melhora a legibilidade e a flexibilidade da criação de objetos. Entretanto, flexibilidade não significa aceitar dados inválidos.

💸 Rodada 6 — O teste vermelho

Horário: 03h45

Prazo da janela de deploy: 04h00

Problema

A suíte de testes apresenta uma falha:

Tests run: 13, Failures: 1

TransferenciaServiceTest.tedDeveCobrarTaxaDe5Reais:
expected: <5.0> but was: <50.0>


O contrato com o cliente estabelece uma taxa de TED de R$ 5,00, mas o serviço utiliza R$ 50,00.

Decisão

Opção B — Investigar o problema e não liberar uma versão incorreta.

Correção
private static final double TAXA_TED = 5.00;


A implementação deve utilizar a taxa definida no contrato e preservar o teste existente.

Em código financeiro de produção, prefira BigDecimal para representar valores monetários.
Plano de ação
Confirmar a regra de negócio da taxa TED.
Investigar a origem do valor incorreto.
Corrigir a implementação.
Executar os testes unitários e de regressão.
Verificar as demais modalidades de transferência.
Se a validação não terminar a tempo, escalar a decisão de lançamento para a liderança.
Aprendizado

Não se deve desabilitar um teste válido nem alterar sua expectativa apenas para acomodar um defeito. A pressão por prazo não elimina a obrigação de respeitar o contrato financeiro.

🧪 Estratégia de testes

O projeto deve incluir testes para:

Transferências com origem e destino corretos.
Preservação dos saldos durante a correção dos extratos.
Criação de contas correntes, poupanças e universitárias.
Rejeição explícita de tipos de conta inválidos.
Construção de objetos com campos opcionais e obrigatórios.
Cálculo correto das taxas de transferência.
Regressão das funcionalidades afetadas pelas correções.
🚀 Validação antes do deploy

Antes de liberar uma versão, executar:

./mvnw clean test


A liberação deve ocorrer somente após a validação dos testes relevantes, a revisão das alterações e a confirmação de que as regras financeiras estão sendo respeitadas.

📚 Conclusão

As rodadas demonstram a importância de quatro princípios fundamentais:

Integridade dos dados: os registros devem representar fielmente as operações realizadas.
Tratamento explícito de erros: falhas devem ser detectadas e comunicadas de forma clara.
Objetos válidos: padrões de construção flexíveis não devem eliminar as validações necessárias.
Testes confiáveis: os testes devem proteger as regras de negócio, e não ser alterados para esconder defeitos.

No contexto bancário, uma solução tecnicamente funcional também precisa ser segura, rastreável e consistente com o que foi prometido ao cliente.
