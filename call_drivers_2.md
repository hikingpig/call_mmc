# register_driver
* Flow:
  - module_init -> (init_function + driver) -> mmc_register_driver

mmc/core/mmc_test.c
```cpp
module_init(mmc_test_init);
```

mmc/core/mmc_test.c
```cpp
// driver is not bus-specific
static struct mmc_driver mmc_driver = {
	.drv		= {
		.name	= "mmc_test",
	},
	.probe		= mmc_test_probe,
	.remove		= mmc_test_remove,
};

static int __init mmc_test_init(void)
{
	return mmc_register_driver(&mmc_driver);
}

```
mmc/core/mmc_test.c
```cpp
static int mmc_test_probe(struct mmc_card *card)
{
	int ret;

	if (!mmc_card_mmc(card) && !mmc_card_sd(card))
		return -ENODEV;

	ret = mmc_test_register_dbgfs_file(card);
	if (ret)
		return ret;

	if (card->ext_csd.cmdq_en) {
		mmc_claim_host(card->host);
		ret = mmc_cmdq_disable(card);
		mmc_release_host(card->host);
		if (ret)
			return ret;
	}

	dev_info(&card->dev, "Card claimed for testing.\n");

	return 0;
}
```

mmc/core/bus.c
```cpp
int mmc_register_driver(struct mmc_driver *drv)
{
	drv->drv.bus = &mmc_bus_type;
	return driver_register(&drv->drv);
}
```

base/driver.c
```cpp
  int driver_register(struct device_driver *drv)
  {
    int ret;
    struct device_driver *other;

    if (!drv->bus->p) {
      pr_err("Driver '%s' was unable to register with bus_type '%s' because the bus was not initialized.\n",
          drv->name, drv->bus->name);
      return -EINVAL;
    }

    if ((drv->bus->probe && drv->probe) ||
        (drv->bus->remove && drv->remove) ||
        (drv->bus->shutdown && drv->shutdown))
      pr_warn("Driver '%s' needs updating - please use "
        "bus_type methods\n", drv->name);

    other = driver_find(drv->name, drv->bus);
    if (other) {
      pr_err("Error: Driver '%s' is already registered, "
        "aborting...\n", drv->name);
      return -EBUSY;
    }

    ret = bus_add_driver(drv);
    if (ret)
      return ret;
    ret = driver_add_groups(drv, drv->groups);
    if (ret) {
      bus_remove_driver(drv);
      return ret;
    }
    kobject_uevent(&drv->p->kobj, KOBJ_ADD);

    return ret;
  }

  EXPORT_SYMBOL_GPL(driver_register);
```

# obtain reference to card

????????????????????????????????????

# call with the card

- from mmc_test_card to mmc_card
```cpp
static int mmc_test_set_blksize(struct mmc_test_card *test, unsigned size)
{
	return mmc_set_blocklen(test->card, size);
}
```
- call a `command` when we have a card
```cpp
int mmc_set_blocklen(struct mmc_card *card, unsigned int blocklen)
{
	struct mmc_command cmd = {};

	if (mmc_card_blockaddr(card) || mmc_card_ddr52(card) ||
	    mmc_card_hs400(card) || mmc_card_hs400es(card))
		return 0;

	cmd.opcode = MMC_SET_BLOCKLEN;
	cmd.arg = blocklen;
	cmd.flags = MMC_RSP_SPI_R1 | MMC_RSP_R1 | MMC_CMD_AC;
	return mmc_wait_for_cmd(card->host, &cmd, 5);
}
```

- call a request
  - device address is 1?
```cpp
static int mmc_test_simple_transfer(struct mmc_test_card *test,
	struct scatterlist *sg, unsigned sg_len, unsigned dev_addr,
	unsigned blocks, unsigned blksz, int write)
{
	struct mmc_request mrq = {};
	struct mmc_command cmd = {};
	struct mmc_command stop = {};
	struct mmc_data data = {};

	mrq.cmd = &cmd;
	mrq.data = &data;
	mrq.stop = &stop;

	mmc_test_prepare_mrq(test, &mrq, sg, sg_len, dev_addr,
		blocks, blksz, write);

	mmc_wait_for_req(test->card->host, &mrq);

	mmc_test_wait_busy(test);

	return mmc_test_check_result(test, &mrq);
}
```

- prepare a request
  - cmd:
    - opcode: MMC_WRITE_MULTIPLE_BLOCK,  MMC_READ_MULTIPLE_BLOCK, MMC_WRITE_BLOCK, MMC_READ_SINGLE_BLOCK
    - arg: dev_addr: 1
    - flags: MMC_RSP_R1 | MMC_CMD_ADTC;
  - we also have data, and maybe need MMC_STOP_TRANSMISSION

```cpp
static void mmc_test_prepare_mrq(struct mmc_test_card *test,
	struct mmc_request *mrq, struct scatterlist *sg, unsigned sg_len,
	unsigned dev_addr, unsigned blocks, unsigned blksz, int write)
{
	if (WARN_ON(!mrq || !mrq->cmd || !mrq->data || !mrq->stop))
		return;

	if (blocks > 1) {
		mrq->cmd->opcode = write ?
			MMC_WRITE_MULTIPLE_BLOCK : MMC_READ_MULTIPLE_BLOCK;
	} else {
		mrq->cmd->opcode = write ?
			MMC_WRITE_BLOCK : MMC_READ_SINGLE_BLOCK;
	}

	mrq->cmd->arg = dev_addr;
	if (!mmc_card_blockaddr(test->card))
		mrq->cmd->arg <<= 9;

	mrq->cmd->flags = MMC_RSP_R1 | MMC_CMD_ADTC;

	if (blocks == 1)
		mrq->stop = NULL;
	else {
		mrq->stop->opcode = MMC_STOP_TRANSMISSION;
		mrq->stop->arg = 0;
		mrq->stop->flags = MMC_RSP_R1B | MMC_CMD_AC;
	}

	mrq->data->blksz = blksz;
	mrq->data->blocks = blocks;
	mrq->data->flags = write ? MMC_DATA_WRITE : MMC_DATA_READ;
	mrq->data->sg = sg;
	mrq->data->sg_len = sg_len;

	mmc_test_prepare_sbc(test, mrq, blocks);

	mmc_set_data_timeout(mrq->data, test->card);
}
```

