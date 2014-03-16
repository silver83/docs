#### intro
raid is about reading and writing from one place (drive C: for example), while underneath - there may be two or more physical drives.

lets say we want to write an image do disk, and the image is made up of 100 blocks.
and lets say we have this drive :

100GB raid drive = 4 x 25GB physical drive @ raid0

raid0 means "striped".
striping is taking each block of the data and putting it in a different drive.
if img was the is an array of such blocks, containing the image data we want to save

#### logical block

	img[0] will go to device0, 
	img[1] to device1
	img[2] to device2
	img[3] to device3
	img[4] to device0 
and so on. 

the general formula is : `img[j]` goes to `device[j % device_count]`.
`j` is the `LOGICAL_BLOCK_INDEX` (logical = spans across discs)

#### physical block

on device3, the img[3] will be considered physical block[0], since its the first block on the physical disk.
on device0, img[4] will be considered physical block[1].

in general the `PHYSICAL_BLOCK_INDEX` = `LOGICAL_BLOCK_INDEX / NUMBER OF DEVICES`.

#### blocks and sectors

each block may contain one or more "sectors". 
each sector - a certain amount of bytes. 

from the code example in raid0.c : 

	#define SECTOR_SIZE 512
	#define SECTORS_PER_BLOCK 2

meanning this is a the block in this case:

	|	  512 bytes		  ,        512 bytes	 |
	|		sector1 	  , 		sector 2 	 |
	|--------- 1024 bytes total in block --------|

#### converting logical to physical blocks and sectors

if we want to read or write, given a sector number i we need to:

1. find the block containing it `(LOGICAL_BLOCK_INDEX)`
2. find the physical device containing the block `(DEVICE_INDEX)`
3. convert block index to a local block index inside the physical device `(PHYSICAL_BLOCK_INDEX)`
4. convert i to an offset inside the physical block `(PHYSICAL_BLOCK_OFFSET)`
5. on the physical device from step 2, seek to the position of the required sector, given by the formula `PHYSICAL_BLOCK_INDEX * BLOCK_SIZE_BYTES + PHYSICAL_BLOCK_OFFSET`
6. read or write what we wanted, **BUT(!!!)** only up to the end of the block. 

if we written or read only a part of the block, we count how many sectors we finished working on, and move to the next sectors we need to work on - go to step 1.

#### EXAMPLE 

	write 75 sectors, starting from sector 49. 
	each block is 5 sectors
	4 devices in raid0
	sector size is 512 bytes
	
1. find block number to start from

		49/5 = 9 = LOGICAL_BLOCK_INDEX			// (start sector) / (sectors in block)
		block 9 contains sector 49.

2. find physical device number 

		9 % 4 = 1 = DEVICE_INDEX				// (LOGICAL_BLOCK_INDEX) % (number of physical devices)

3. convert block index to a local block index inside the physical device

		9 / 4 = 2 = PHYSICAL_BLOCK_INDEX 		// (LOGICAL_BLOCK_INDEX) / (number of physical devices)

4. convert sector number (i) to a sector offset inside the physical block

		49 % 5 = 4 = PHYSICAL_BLOCK_OFFSET		// (start sector) % (sectors in block)

5. on device1, seek to correct position

		physicalOffset = (2 * 5 + 4) * 512		// (PHYSICAL_BLOCK_INDEX * SECTORS_PER_BLOCK + PHYSICAL_BLOCK_OFFSET) * SECTOR_SIZE
	
		lseek(devices[1], physicalOffset, SEEK_SET)

6. read or write what we wanted - we can only go up until the end of the current block.
	if we start at the 4th out of 5 sectors, we are allowed to only read 1 more...

		sectorsToRead = 5 - 4					// SECTORS_PER_BLOCK - PHYSICAL_BLOCK_OFFSET
		size = sectorsToRead * SECTOR_SIZE
		read(devices[DEVICE_INDEX], buf, size))



#### where in the code each step is :

	void do_raid0_rw(char* operation, int sector, int count)
	{
		int i;
		for (i = sector; i < sector+count; ) {
			
			int block_num = i / SECTORS_PER_BLOCK; // step 1
			int dev_num = block_num % num_dev;	    // step 2
	
			// make sure device didn't fail
			if (dev_fd[dev_num] < 0) {
				printf("Operation on bad device %d\n", dev_num);
				break;
			}
	
			// find offset of sector inside device
			int block_start = i / (num_dev * SECTORS_PER_BLOCK); // step 3 
			int block_off = i % SECTORS_PER_BLOCK;               // step 4
			int sector_start = block_start * SECTORS_PER_BLOCK + block_off;
			int offset = sector_start * SECTOR_SIZE;
			
			// try to write few sectors at once
			int num_sectors = SECTORS_PER_BLOCK - block_off;
			while (i+num_sectors > sector+count)
				--num_sectors;
			int sector_end = sector_start + num_sectors - 1;
			int size = num_sectors * SECTOR_SIZE;                // step 6 part1
			
			// validate calculations
			assert(num_sectors > 0);
			assert(size <= BUFFER_SIZE);
			assert(offset+size <= DEVICE_SIZE);
			
			// seek in relevant device
			assert(offset == lseek(dev_fd[dev_num], offset, SEEK_SET)); //step 5

			// step 6 part2			
			if (!strcmp(operation, "READ"))		
				assert(size == read(dev_fd[dev_num], buf, size));
			else if (!strcmp(operation, "WRITE"))
				assert(size == write(dev_fd[dev_num], buf, size));
			
			printf("Operation on device %d, sector %d-%d\n",
				dev_num, sector_start, sector_end);
			
			i += num_sectors;
		}
	}