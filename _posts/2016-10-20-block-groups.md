---
layout: post
title:  "Nova Block Groups"
date:   2016-10-20 18:14:54
categories: NOVA
tags: Nova filesystem PM NVM
---

Nova Block Groups
=======



## In  super.c, change cpus number.

```c
static inline void set_default_opts(struct nova_sb_info *sbi)
{                       
        set_opt(sbi->s_mount_opt, HUGEIOREMAP);
        set_opt(sbi->s_mount_opt, ERRORS_CONT);
        sbi->reserved_blocks = RESERVED_BLOCKS;
        //change by wen pan: add cpus to use more block groups,
        // lists to improve paralellism
        sbi->cpus = 1024;
        //sbi->cpus = num_online_cpus();
        sbi->map_id = 0;
}
```
## Free list is got by

```c
struct free_list *free_list = nova_get_free_list(sb, i); 
```
## Treate list as block group 

+ Add a counter (*alloc_count* ) to  indicate how many times that the blocks in this group has been allocated. 
   + Sorted link list to decide which group should be use.  

- ** *nova_new_blocks( )*  might *nova_get_candidate_free_list( )* to decide which  
  Free list should ot use. **
  When current free_list[cpuid] can't satisfies requirements, find an aternative free_list
```c

static int nova_get_candidate_free_list(struct super_block *sb) 
 {
 	cpuid = smp_processor_id(); // get the current CPU ID.
 	...
 	...
 	free_list = nova_get_free_list(sb, cpuid);
	if (free_list->num_free_blocks < num_blocks || !free_list->first_node) {
                nova_dbgv("%s: cpu %d, free_blocks %lu, required %lu, "
                        "blocknode %lu\n", __func__, cpuid,
                        free_list->num_free_blocks, num_blocks,
                        free_list->num_blocknode); 
                if (free_list->num_free_blocks >= num_blocks) {
                        nova_dbg("first node is NULL "
                                "but still has free blocks\n");
                        temp = rb_first(&free_list->block_free_tree);
                        first = container_of(temp, struct nova_range_node, node);
                        free_list->first_node = first;
                } else {
                        spin_unlock(&free_list->s_lock);
                        if (retried >= 3)
                                return -ENOMEM;
                        /* Find out the free list with most free blocks */ 
                        cpuid = nova_get_candidate_free_list(sb);
                        retried++;
                        goto retry;
                }
        }
	}

 }
```
- ** current cpuid is got by *cpuid = smp_processor_id()*, and it's used for: **
  + allocate free page
  + Journal transaction

- ** Allocate blocks from *free_list* **
  Ignore superpage support, then wile loop is not necessary.
  Allocate from current node, if not enough blks in this node, allocate all of them and then goto next.
  Retuen number of allocated pages
```c
static unsigned long nova_alloc_blocks_in_free_list(struct super_block *sb,
        struct free_list *free_list, unsigned short btype,
        unsigned long num_blocks, unsigned long *new_blocknr)
{
        ...

        tree = &(free_list->block_free_tree);
        temp = &(free_list->first_node->node);

        while (temp) {
                step++;
                curr = container_of(temp, struct nova_range_node, node);

                curr_blocks = curr->range_high - curr->range_low + 1;
						...
			
                        /* Otherwise, allocate the whole blocknode */
                        if (curr == free_list->first_node) {
                                next_node = rb_next(temp);
                                if (next_node)
                                        next = container_of(next_node,
                                                struct nova_range_node, node);
                                free_list->first_node = next;
                        }

                        rb_erase(&curr->node, tree);
                        free_list->num_blocknode--;
                        num_blocks = curr_blocks;
                        *new_blocknr = curr->range_low;
                        nova_free_blocknode(sb, curr);
                        found = 1;
                        break;
                }

                /* Allocate partial blocknode */
                *new_blocknr = curr->range_low;
                curr->range_low += num_blocks;
                found = 1;
                break;
        }

        free_list->num_free_blocks -= num_blocks;
  alloc_steps += step;

        if (found == 0)
                return -ENOSPC;

        return num_blocks;
}
```
## Allocate space for free_list (use *kzalloc*)

This function is called by *nova_fill_super*

```c
int nova_alloc_block_free_lists(struct super_block *sb){
	...
        sbi->free_lists = kzalloc(sbi->cpus * sizeof(struct free_list),
                                                        GFP_KERNEL);
    ...                                                  

        for (i = 0; i < sbi->cpus; i++) {
                free_list = nova_get_free_list(sb, i);
                free_list->block_free_tree = RB_ROOT;
                spin_lock_init(&free_list->s_lock);
        }                       
    ...
}
```
