Sempre pode suas redes

Sério, sempre pode suas redes. Docker networks abandonadas, desligadas de qualquer container podem destruir sua internet.

Eu fiquei duas semanas com problema na internet da minha máquina: por algum motivo, na rede de casa, a conexão do ubuntu ficava falhando.

Hora penava pra resolver DNSs, hora parava de funcionar completamente. Mas às vezes \(geralmente quando eu pedia ajuda\) ela funcionava.

Tentei desabilitar o IPV6 nas configurações de rede, desinstalar o `systemd-resolved`, mexer com o network manager, editar o /etc/resolv.conf diretamente.

Nada.

Em outras redes, na casa de amigos, na cafeteria, a internet funcionava. Eu cheguei a achar que meu provedor estava de sacanagem.

Até que tive que ficar num airBnb, e o problema voltou.

Olhei pro roteador, e era o mesmo `d-link` que eu tinha em casa. Já comecei a desenvolver uma crendice.

Acabou que não pude ir até o fim, pois um dia rodei um `docker network ls` e notei que tinha muita coisa na listagem.

        docker network prune
        >Yes

E problema resolvido.

Se alguém souber mais sobre isso, ou como reproduzir o problema, por favor, [entre em contato](mailto:gui.garcia67@gmail.com).

Mas lembre-se: sempre pode suas redes.

Tags: docker, dns, networking
