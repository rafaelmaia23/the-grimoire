# INC: KVM indisponível — SVM desativado na BIOS (B550M Aorus Elite)

- **Data:** 2026-05-22
- **Local:** PC pessoal
- **Sistema:** Fedora 44, kernel 7.0.9, Ryzen 5 5700X, Gigabyte B550M Aorus Elite

---

## Contexto / Sintoma inicial

Após instalar a stack `@virtualization` e habilitar o `libvirtd`, ao abrir o virt-manager e tentar criar uma nova VM, apareceu um aviso amarelo:

> **Aviso:** KVM não está disponível. Isso pode significar que o pacote KVM não está instalado ou os módulos KVM do Kernel não estão carregados. Suas máquinas virtuais podem executar em baixo desempenho.

Sem aceleração por hardware, as VMs caem em emulação pura via TCG/QEMU — uma VM Linux mínima leva minutos pra bootar. Inaceitável.

Hardware confirmado compatível: Ryzen 5 5700X tem AMD-V (visível pela flag `svm` no `/proc/cpuinfo`). A RX 6600 é irrelevante pra virtualização de CPU.

---

## Diagnóstico

### Estado do módulo

```bash
lsmod | grep -E '^kvm'
# (vazio)

ls -la /dev/kvm
# ls: não foi possível acessar '/dev/kvm': Arquivo ou diretório inexistente
```

Módulo `kvm_amd` não carregado, device `/dev/kvm` inexistente. Sem ele, nada de aceleração.

### Validação detalhada

```bash
sudo virt-host-validate qemu
# QEMU: A verificar por virtualização de hardware : FALHA
# (Máquina não compatível com KVM; Funcionalidades da CPU de virtualização HW
#  não encontradas. Apenas CPUs emuladas estão disponíveis;
#  a performance será significativamente limitada)
```

### Tentativa de carregar manualmente

```bash
sudo modprobe kvm     # OK, módulo genérico carrega
sudo modprobe kvm_amd
# modprobe: ERROR: could not insert 'kvm_amd': Operation not supported
```

**Esta é a smoking gun.** O módulo `kvm` (genérico) carrega, mas `kvm_amd` é rejeitado pelo hardware no momento da inserção. Isso indica que o CPU **suporta** SVM (por isso a flag aparece no `cpuinfo`) mas o **bit de habilitação está bloqueado** — geralmente por configuração de BIOS.

### Confirmação definitiva via dmesg

```bash
sudo dmesg | grep -iE 'kvm|svm' | tail -10
# [    0.394027] SVM disabled (by BIOS) in MSR_VM_CR
# [  102006.759] kvm_amd: SVM not supported by CPU 6
```

A mensagem `SVM disabled (by BIOS) in MSR_VM_CR` é literal: o kernel leu o registrador `MSR_VM_CR` do CPU e encontrou o bit que indica "SVM bloqueado pela BIOS". Não há contorno via software.

---

## Causa Raiz

A placa Gigabyte B550M Aorus Elite vem com **SVM Mode desabilitado por padrão de fábrica**. Não houve update de BIOS recente — o setup anterior, onde "funcionava direto", provavelmente foi feito após habilitar SVM manualmente e essa configuração foi esquecida ou perdida em algum momento (Clear CMOS, reset, ou nunca foi feita e a confusão era outra).

Reforço da hipótese: a BIOS atual é `FF` (datada de 09/26/2025), placa vinda direto de uma instalação limpa do Fedora 44, e o sintoma é exatamente `disabled (by BIOS)`.

> **Lição:** flag `svm` no `/proc/cpuinfo` indica suporte do CPU, **não habilitação**. Pra confirmar de verdade, olhar `/dev/kvm` e `dmesg`.

---

## Solução

### Habilitar SVM na BIOS

1. Reboot, entrar na BIOS (`Del` no boot)
2. Se cair em "Easy Mode", `F2` pra Advanced Mode
3. Aba **Tweaker** → **Advanced CPU Settings**
4. **SVM Mode** → **Enabled**
5. `F10` → salvar e reiniciar

> Em algumas versões da BIOS Gigabyte, o caminho é `Settings → Miscellaneous → AMD CBS → CPU Common Options`. O nome do setting é sempre **SVM Mode** ou apenas **SVM**.

IOMMU não precisa ser ativado para uso comum com NAT — só é necessário pra PCI passthrough (passar GPU pra dentro de uma VM). Pode deixar quieto.

### Verificação pós-reboot

```bash
# dmesg não mostra mais "SVM disabled"
sudo dmesg | grep -iE 'kvm|svm' | tail -10
# [   21.402715] kvm_amd: TSC scaling supported
# [   21.402718] kvm_amd: Nested Virtualization enabled
# [   21.402720] kvm_amd: Nested Paging enabled
# [   21.402721] kvm_amd: LBR virtualization supported
# [   21.402724] kvm_amd: Virtual VMLOAD VMSAVE supported
# [   21.402725] kvm_amd: Virtual GIF supported

# /dev/kvm existe com grupo kvm
ls -la /dev/kvm
# crw-rw-rw-. 1 root kvm 10, 232 May 22 00:14 /dev/kvm

# Módulos carregados
lsmod | grep kvm
# kvm_amd               270336  0
# kvm                  1552384  1 kvm_amd
# irqbypass              16384  1 kvm

# Validação completa
sudo virt-host-validate qemu
# QEMU: A verificar por virtualização de hardware : PASSAR (SVM)
# QEMU: A verificar se dispositivo '/dev/kvm' existe : PASSAR
# QEMU: A verificar se dispositivo '/dev/kvm' está acessível : PASSAR
# QEMU: A verificar por suporte IOMMU : PASSAR (IVRS)
# QEMU: A verificar se IOMMU está activado pelo kernel : PASSAR
# (warning em SEV/SEV-ES/TDX é esperado e irrelevante)
```

Após reboot, virt-manager não mostra mais o aviso amarelo e VMs criadas usam aceleração KVM normal.

---

## Estado final

```
SVM Mode (BIOS)  : Enabled
Módulo kvm_amd   : carregado, com Nested Virtualization, Nested Paging, etc.
/dev/kvm         : existe, root:kvm, permissão 660
virt-host-validate: tudo PASSAR exceto warning informativo de SEV
virt-manager     : aceleração KVM funcionando, sem aviso amarelo
```

---

## Como evitar no futuro

- **Em qualquer setup novo de PC com placa AMD**, verificar SVM na BIOS **antes** de instalar libvirt. Pra placas Gigabyte (B550, X570, B650 etc.), o default de fábrica é **Disabled**.
- **Diagnóstico definitivo** quando virt-manager reclamar de KVM:
  ```bash
  sudo dmesg | grep -iE 'kvm|svm' | tail
  ```
  Se aparecer `SVM disabled (by BIOS) in MSR_VM_CR` (AMD) ou `VMX disabled by BIOS` (Intel), é configuração da BIOS — sem fix via software.
- **Após qualquer Clear CMOS, troca de bateria CMOS ou update de BIOS**, revalidar SVM antes de assumir que o ambiente de virtualização continua funcional. Um simples `ls /dev/kvm` no boot já confirma.

---

## Referência

- Setup completo: `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md`
- Documentação Arch sobre KVM: https://wiki.archlinux.org/title/KVM#Checking_support_for_KVM
