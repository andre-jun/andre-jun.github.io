---
title: "Segundo patch no kernel Linux: Usando 'guard' em vez de lock+unlock manuais"
date: 2026-04-21 15:18:11 -0300
categories: [Kernel, Patches]
tags: [kernel, linux, amdgpu, drm, patch, contribuição]
---

Neste post vou documentar o segundo patch que enviei para o kernel Linux,
feito em conjunto com Gabriel Dimant e Guilherme Gabriel na disciplina
MAC0470, Desenvolvimento de Software Livre da USP.

## O subsistema

O driver escolhido foi o **dpm**, o driver de GPU AMD presente no kernel
Linux, dentro do subsistema DRM (`drivers/gpu/drm/amd/pm/`). A escolha
foi, em parte, recomendada pelo professor, que propôs a substituição de
`mutex_lock` e `mutex_unlock` pelo mais moderno `guard`.

### A escolha do arquivo

Para escolher o arquivo em que executaríamos a troca de lock e unlock
manuais por `guard`, fizemos um grep na raiz da árvore do kernel que
seleciona os arquivos com `mutex_lock` e os ordena por número de ocorrências:

```bash
grep -rn "mutex_lock" | awk -F: '{print $1}' | sort | uniq -c | sort -nr | awk '{print $2, $1}'
```

O arquivo com o maior número de ocorrências é o `drivers/usb/gadget/function/uvc_configfs.c`, com 124 ocorrências. Porém,
ao consultar o arquivo de mantenedores e observar que, no momento, não há um mantenedor para esse arquivo (unsupported),
o grupo decidiu escolher o arquivo com o segundo maior número de ocorrências: o
`drivers/gpu/drm/amd/pm/amdgpu_dpm.c`, com 103 ocorrências.

## O que encontramos

Navegando pelo código, identificamos três padrões de situações em que haveria a troca de `mutex` por `guard`.

A primeira ocorre quando o mutex é usado para proteger uma chamada simples antes do retorno da função:
```
@@ -46,10 +47,9 @@ int amdgpu_dpm_get_sclk(struct amdgpu_device *adev, bool low)
       if (!pp_funcs->get_sclk)
               return 0;

       mutex_lock(&adev->pm.mutex);
       ret = pp_funcs->get_sclk((adev)->powerplay.pp_handle,
                                low);
       mutex_unlock(&adev->pm.mutex);

       return ret;
}
```

A segunda ocorre quando o mutex segue o padrão com `goto`:
```
@@ -80,13 +79,12 @@ int amdgpu_dpm_set_powergating_by_smu(struct amdgpu_device *adev,
       enum ip_power_state pwr_state = gate ? POWER_STATE_OFF : POWER_STATE_ON;
       bool is_vcn = block_type == AMD_IP_BLOCK_TYPE_VCN;

       mutex_lock(&adev->pm.mutex);

       if (atomic_read(&adev->pm.pwr_state[block_type]) == pwr_state &&
                       (!is_vcn || adev->vcn.num_vcn_inst == 1)) {
               dev_dbg(adev->dev, "IP block%d already in the target %s state!",
                               block_type, gate ? "gate" : "ungate");
               goto out_unlock;
       }

       switch (block_type) {
@@ -115,9 +113,6 @@ int amdgpu_dpm_set_powergating_by_smu(struct amdgpu_device *adev,
       if (!ret)
               atomic_set(&adev->pm.pwr_state[block_type], pwr_state);

out_unlock:
       mutex_unlock(&adev->pm.mutex);

       return ret;
}
```

E a terceira ocorre quando o mutex é liberado antes do fim da função:
```
@@ -126,9 +121,9 @@ int amdgpu_dpm_set_gfx_power_up_by_imu(struct amdgpu_device *adev)
       struct smu_context *smu = adev->powerplay.pp_handle;
       int ret = -EOPNOTSUPP;

       mutex_lock(&adev->pm.mutex);
       ret = smu_set_gfx_power_up_by_imu(smu);
       mutex_unlock(&adev->pm.mutex);

       msleep(10);
```

Todos os três padrões foram relativamente fáceis de corrigir, sendo que apenas o terceiro precisou de `scoped_guard`, e o segundo exigiu um pouco
mais de atenção para garantir que não houvesse alteração de funcionalidade.

## Dificuldades no caminho

A única dificuldade encontrada pelo grupo foi na função `int amdgpu_dpm_force_performance_level`, na qual encontramos um uso de
mutex que não soubemos como adaptar. No final, optamos por não modificar essa função.
```
...
else if ((current_level & profile_mode_mask) &&
		 !(level & profile_mode_mask))
		amdgpu_dpm_exit_umd_state(adev);

	mutex_lock(&adev->pm.mutex);

	if (pp_funcs->force_performance_level(adev->powerplay.pp_handle,
					      level)) {
		mutex_unlock(&adev->pm.mutex);
		/* If new level failed, retain the umd state as before */
		if (!(current_level & profile_mode_mask) &&
		    (level & profile_mode_mask))
			amdgpu_dpm_exit_umd_state(adev);
		else if ((current_level & profile_mode_mask) &&
			 !(level & profile_mode_mask))
			amdgpu_dpm_enter_umd_state(adev);

		return -EINVAL;
	}

	adev->pm.dpm.forced_level = level;

	mutex_unlock(&adev->pm.mutex);

	return 0;
}
```

## Divisão do trabalho

O patch foi desenvolvido em conjunto com Gabriel Dimant e Guilherme Gabriel.
A discussão sobre a abordagem correta e a implementação foram feitas em grupo.
O envio ficou comigo, com os três listados no commit via `Co-developed-by`.

## O patch

A mensagem de commit ficou assim:

```
drm/amd/pm: Use guard(mutex) instead of manual lock+unlock

Use guard() and scoped_guard() for handling mutex lock instead of
manually locking and unlocking the mutex. This prevents forgotten
locks due to early exits and removes the need of gotos.

Signed-off-by: Andre Jun Hirata <andrejhirata@usp.br>
Co-developed-by: Gabriel Dimant <gabriel.dimant@usp.br>
Signed-off-by: Gabriel Dimant <gabriel.dimant@usp.br>
Co-developed-by: Guilherme Gabriel <guilhermesangabriel@usp.br>
Signed-off-by: Guilherme Gabriel <guilhermesangabriel@usp.br>

```

## Envio

O patch foi enviado ao manteinedor e para a mailing list do subsistema amdgpu:

- **Maintainers:** Kenneth Feng
- **Lista:** `amd-gfx@lists.freedesktop.org`

o patch pode ser encontrado no [lore.kernel](https://lore.kernel.org/amd-gfx/20260421015506.9230-1-andrejhirata@usp.br/T/#u)

## Resultado

O patch foi enviado mas ainda nao recebemos feedback.
