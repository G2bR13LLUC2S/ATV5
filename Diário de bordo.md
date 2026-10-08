Diário de Bordo — Diagnóstico de Escalonamento de Processos e Chamadas de Sistema

1. Introdução

Este diário de bordo tem como objetivo analisar o funcionamento do Sistema Operacional a partir do estudo de caso da startup CloudData. A empresa utiliza um servidor de núcleo único para executar dois tipos de tarefas: processos interativos, responsáveis por atender às requisições dos usuários da interface web, e processos batch, responsáveis pela geração de relatórios financeiros.

O servidor utiliza o algoritmo de escalonamento FCFS (First-Come, First-Served). Entretanto, os usuários relatam que a interface web apresenta congelamentos durante vários segundos. Para compreender esse problema, serão analisadas as chamadas de sistema, a mudança entre modo usuário e modo kernel, o funcionamento dos algoritmos de escalonamento e os mecanismos para evitar a inanição de processos.

A pesquisa utiliza documentação técnica, material audiovisual e conteúdo em áudio relacionados a Sistemas Operacionais.

2. Parte A — Entendendo a Barreira do Sistema: Chamadas de Sistema

2.1. Como o processo solicita a leitura dos dados financeiros ao Sistema Operacional?

Um processo executado em modo usuário não possui permissão para acessar diretamente os dispositivos de hardware de maneira irrestrita. Por isso, quando o processo de geração de relatórios precisa ler os dados financeiros armazenados em disco, ele solicita esse serviço ao Sistema Operacional por meio de uma chamada de sistema (system call).

No Linux, um exemplo dessa operação é a chamada "read()", utilizada para tentar ler dados de um arquivo ou recurso identificado por um descritor de arquivo e transferi-los para uma área de memória indicada pelo programa (LINUX MAN-PAGES PROJECT, [s.d.]).

O funcionamento pode ser descrito pelas seguintes etapas:

1. O processo de geração de relatórios solicita a leitura dos dados.
2. O programa realiza uma chamada de sistema, como "read()".
3. O processador transfere o controle para o kernel por meio do mecanismo apropriado de chamada de sistema.
4. O Sistema Operacional verifica e gerencia a operação de leitura.
5. Os dados são disponibilizados ao programa quando a operação pode ser concluída, e o processo continua sua execução.

A leitura nem sempre exige um acesso físico ao disco, pois os dados podem estar disponíveis em um cache de memória. Caso seja necessário aguardar uma operação de entrada e saída, o processo pode ficar bloqueado até que os dados estejam disponíveis.

Dessa forma, as chamadas de sistema funcionam como uma interface controlada entre as aplicações e os serviços oferecidos pelo kernel.

2.2. O que ocorre durante a mudança entre modo usuário e modo kernel?

Os processadores modernos possuem níveis de privilégio que ajudam a proteger os recursos do sistema. Dois conceitos importantes nesse contexto são o modo usuário e o modo kernel.

Modo usuário: é o ambiente em que normalmente são executadas as aplicações. Nesse modo, o programa possui acesso restrito às operações privilegiadas do processador e aos recursos protegidos do sistema.

Modo kernel: é o modo privilegiado utilizado pelo núcleo do Sistema Operacional para executar operações que exigem permissões especiais, como gerenciar dispositivos, memória e recursos de entrada e saída.

Quando uma aplicação realiza uma chamada de sistema, ocorre uma transição controlada para o modo kernel. O kernel processa a solicitação e, quando a chamada termina, o resultado é devolvido à aplicação, que pode continuar em modo usuário.

É importante destacar que a mudança de modo não significa necessariamente uma troca de processo. A chamada pode ser executada pelo mesmo processo, que passa temporariamente a utilizar os serviços privilegiados do Sistema Operacional.

Se a operação de leitura precisar aguardar o disco, o processo poderá ficar bloqueado. Nesse caso, o escalonador poderá selecionar outro processo que esteja pronto para utilizar a CPU.

Essa separação entre os modos de execução contribui para a proteção do hardware e para o controle de acesso aos recursos do sistema (ARPACI-DUSSEAU; ARPACI-DUSSEAU, [s.d.]).

3. Parte B — Diagnosticando o Escalonador

3.1. Por que o FCFS está causando o congelamento da interface web?

O FCFS (First-Come, First-Served), também conhecido como FIFO (First-In, First-Out), seleciona os processos prontos conforme a ordem de chegada. Na sua versão clássica, não considera a duração estimada de cada tarefa para definir quem será executado primeiro.

No cenário da CloudData, um processo de geração de relatórios pode exigir um período prolongado de processamento da CPU. Se uma requisição web chegar enquanto esse processo estiver executando, ela poderá precisar aguardar sua vez.

Mesmo que a requisição web precise de pouco tempo de CPU, o usuário poderá perceber uma demora significativa na resposta da interface.

Esse comportamento está relacionado ao efeito comboio (convoy effect), que ocorre quando tarefas menores ficam esperando atrás de uma tarefa mais longa.

Entretanto, é necessário considerar que os relatórios também realizam operações de entrada e saída. Quando um processo fica bloqueado aguardando dados do disco, ele deixa de utilizar a CPU, permitindo que outro processo pronto seja selecionado. Portanto, o problema não deve ser explicado como se um processo bloqueado monopolizasse a CPU durante toda a operação de leitura.

A demora da interface pode resultar de períodos prolongados de processamento, filas de espera, contenção por recursos ou latência nas operações de entrada e saída. O FCFS pode agravar o tempo de resposta das requisições interativas por não priorizar tarefas curtas ou sensíveis à latência (ARPACI-DUSSEAU; ARPACI-DUSSEAU, [s.d.]).

3.2. O que significa dizer que o FCFS é não preemptivo?

Um algoritmo de escalonamento não preemptivo não retira a CPU de um processo em execução apenas porque outro processo chegou e precisa ser atendido.

No FCFS clássico, o processo continua executando até concluir sua utilização da CPU ou bloquear-se, por exemplo, para aguardar uma operação de entrada e saída.

Isso significa que uma requisição web recém-chegada pode precisar esperar que o processo anterior termine sua utilização da CPU ou fique bloqueado. Como consequência, o tempo de resposta pode aumentar.

Em contraste, um algoritmo preemptivo permite interromper um processo em execução para selecionar outro processo pronto, conforme as regras do escalonador.

Essa característica é importante para a CloudData porque as requisições web precisam de respostas rápidas, enquanto os relatórios financeiros podem tolerar períodos maiores de espera.

Assim, a adoção de um algoritmo preemptivo pode melhorar a distribuição do tempo de CPU entre os processos, embora o resultado também dependa do quantum, das prioridades e do comportamento das operações de entrada e saída (ARPACI-DUSSEAU; ARPACI-DUSSEAU, [s.d.]).

4. Parte C — Propondo a Solução

4.1. Qual algoritmo é mais adequado para resolver o problema da CloudData?

Os três algoritmos apresentados na atividade possuem características diferentes.

SJF — Shortest Job First

O SJF seleciona o processo pronto com a menor duração estimada de CPU. Na versão não preemptiva, depois que o processo começa a executar, ele continua até terminar sua utilização da CPU ou bloquear-se.

Esse algoritmo pode reduzir o tempo médio de espera em determinadas condições, mas depende de estimativas da duração das tarefas. Além disso, uma requisição curta que chegue enquanto uma tarefa longa está executando não poderá interrompê-la na versão não preemptiva.

SRTN — Shortest Remaining Time Next

O SRTN, também conhecido como SRTF (Shortest Remaining Time First), é a versão preemptiva do SJF. Ele seleciona o processo com o menor tempo restante estimado de CPU.

Se chegar um processo cujo tempo estimado de execução seja menor que o tempo restante do processo atual, o escalonador poderá interromper o processo em execução e selecionar o novo.

Essa estratégia favorece tarefas curtas, mas depende de estimativas de tempo restante e pode fazer tarefas longas esperarem por muito tempo quando novas tarefas curtas continuam chegando.

Round-Robin (RR)

O Round-Robin organiza os processos prontos em uma fila circular e concede a cada um um intervalo de utilização da CPU chamado quantum.

Quando o quantum termina, o processo pode ser interrompido e colocado novamente no fim da fila de prontos, permitindo que outro processo utilize a CPU.

O tamanho do quantum influencia o comportamento do algoritmo. Um intervalo muito longo pode aumentar o tempo de resposta dos processos interativos, enquanto um intervalo muito curto pode elevar o custo das trocas de contexto.

Solução recomendada

Para o cenário apresentado, recomenda-se o Round-Robin, considerando que a prioridade é melhorar a responsividade da interface web e que não foram fornecidas estimativas confiáveis da duração de cada tarefa.

O Round-Robin permite que os processos prontos recebam oportunidades de utilizar a CPU sem que uma tarefa longa precise necessariamente terminar para que outras tarefas executem. Isso é adequado para ambientes interativos em que a alternância entre processos é importante.

Entretanto, o Round-Robin convencional não garante, sozinho, prioridade máxima para a interface web. Uma solução mais adequada para a CloudData seria combinar filas de prioridade por tipo de tarefa com Round-Robin entre processos de mesma prioridade.

Nesse modelo, as requisições interativas poderiam receber prioridade maior, enquanto os relatórios financeiros continuariam recebendo oportunidades de execução. Seria necessário configurar as prioridades e o quantum de acordo com os requisitos do servidor.

A empresa também deveria monitorar o tempo de resposta da interface, o tempo de espera dos relatórios, a utilização da CPU e a latência das operações de disco para verificar se a alteração resolveu o problema.

4.2. O que é Starvation e como evitar esse problema?

Starvation, ou inanição, ocorre quando um processo permanece esperando indefinidamente porque outros processos continuam recebendo preferência para executar.

Na CloudData, esse problema poderia ocorrer se as requisições web recebessem sempre prioridade máxima e novas requisições continuassem chegando. Se o escalonador nunca permitisse que os processos de relatório recebessem CPU, as tarefas batch poderiam ficar esperando indefinidamente.

Uma técnica para reduzir esse risco é o aging, ou envelhecimento de prioridade.

O aging aumenta gradualmente a prioridade de um processo conforme aumenta seu tempo de espera. Dessa forma, os processos que permanecem muito tempo na fila passam a ter mais chances de serem selecionados pelo escalonador.

Outra possibilidade é reservar uma parcela do tempo de CPU para os processos batch, evitando que sejam completamente privados de execução.

No cenário da CloudData, a combinação de prioridades para favorecer a interface web com um mecanismo de aging para os relatórios permitiria buscar um equilíbrio entre responsividade e justiça no compartilhamento da CPU.

É importante observar que o aging reduz o risco de inanição, mas a garantia de progresso depende das regras de escalonamento adotadas e de como as prioridades são atualizadas (ARPACI-DUSSEAU; ARPACI-DUSSEAU, [s.d.]).

5. Diagrama de síntese

O diagrama abaixo relaciona as chamadas de sistema, a leitura de dados, o escalonamento e a solução proposta para a CloudData.

flowchart TD
    A[Processos da CloudData] --> B[Processo interativo]
    A --> C[Processo batch: relatório]

    C --> D[Solicitação de leitura: read]
    D --> E[Transição para modo kernel]
    E --> F[Gerenciamento da operação de E/S]
    F --> G{Dados disponíveis?}
    G -- Não --> H[Processo bloqueado]
    H --> I[Escalonador seleciona outro processo pronto]
    G -- Sim --> J[Retorno ao processo]

    B --> K[Fila de processos prontos]
    I --> K
    J --> K

    K --> L[FCFS: ordem de chegada]
    L --> M[Possível aumento do tempo de resposta]

    K --> N[Round-Robin: quantum de CPU]
    N --> O[Alternância entre processos prontos]
    O --> P[Melhoria potencial da responsividade]

    P --> Q[Prioridades para tarefas interativas]
    Q --> R[Aging para reduzir starvation]
    R --> S[Equilíbrio entre responsividade e progresso dos relatórios]

Interpretação: as chamadas de sistema permitem solicitar operações ao kernel. Quando uma tarefa fica bloqueada por entrada e saída, o escalonador pode selecionar outro processo pronto. A escolha do algoritmo influencia a distribuição do tempo de CPU, e a combinação de prioridades com envelhecimento pode favorecer a interface sem deixar os relatórios indefinidamente sem execução.

6. Considerações finais

A análise do caso da CloudData demonstra que o escalonamento de processos influencia diretamente a capacidade de resposta de um servidor que atende tarefas interativas e de processamento em lote.

As chamadas de sistema fornecem uma interface controlada entre as aplicações e o kernel. Durante uma operação de entrada e saída, um processo pode ficar bloqueado, permitindo que outro utilize a CPU.

O FCFS é simples, mas pode prejudicar a responsividade quando tarefas longas estão à frente de tarefas interativas na fila. O Round-Robin representa uma alternativa adequada para melhorar a alternância entre processos prontos. Para dar preferência às requisições web, pode ser combinado com prioridades.

Por fim, o uso de aging ou a reserva de tempo de CPU para processos batch pode reduzir o risco de inanição. A solução deve ser avaliada por meio de métricas de tempo de resposta, espera e utilização dos recursos, pois o escalonamento não é necessariamente a única causa de lentidão da aplicação.

7. Referências

ARPACI-DUSSEAU, Remzi H.; ARPACI-DUSSEAU, Andrea C. Operating Systems: Three Easy Pieces. Madison: Arpaci-Dusseau Books, [s.d.]. Disponível em: https://pages.cs.wisc.edu/~remzi/OSTEP/. Acesso em: 8 out. 2026.

CAFÉ DEBUG. #167 Threads, Paralelismo e SO na Prática para Devs. [S. l.], 14 jul. 2025. Podcast. Disponível em: https://podcasts.apple.com/br/podcast/167-threads-paralelismo-e-so-na-pr%C3%A1tica-para-devs/id1367730836?i=1000717118967. Acesso em: 8 out. 2026.

INTRODUCTION TO OPERATING SYSTEMS. CPU Scheduling Algorithms. YouTube, 14 ago. 2016. Vídeo. Disponível em: https://www.youtube.com/watch?v=4hCih9eLc7M. Acesso em: 8 out. 2026.

LINUX MAN-PAGES PROJECT. read(2) — Linux manual page. [S. l.], [s.d.]. Disponível em: https://www.man7.org/linux/man-pages/man2/read.2.html. Acesso em: 8 out. 2026.

ROCHA, Leonardo. Sistemas operacionais: o modelo de processos. [Material de aula, Aula 7]. [S. l.: s. n.], [s. d.].

8. Registro da pesquisa multimídia

- Texto: documentação do Linux sobre "read()" e livro Operating Systems: Three Easy Pieces, utilizados para fundamentar as chamadas de sistema, os processos e o escalonamento.
- Vídeo: aula CPU Scheduling Algorithms, publicada pelo canal Introduction to Operating Systems, selecionada para aprofundar FCFS, SJF e Round-Robin.
- Áudio: episódio 167 do podcast Café Debug, sobre threads, paralelismo e Sistemas Operacionais, selecionado como complemento para compreender concorrência e desempenho.

Observação: antes da entrega, confirme que o vídeo e o podcast foram efetivamente acessados e consultados. As referências acima identificam os materiais selecionados para a pesquisa; não se deve afirmar que um conteúdo foi integralmente assistido ou ouvido sem que isso tenha ocorrido.
