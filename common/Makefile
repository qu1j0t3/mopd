#simple make file for linux


OBJS=cmp.o dl.o get.o mopdef.o  pf-linux2.o put.o device.o file.o loop-linux2.o nma.o print.o rc.o

libcommon.a: ${OBJS}
	 ar -rs libcommon.a ${OBJS}

.c.o: .o
	cc -c $(CFLAGS) $<
	
clean: 
	rm -f *.o *.a *~
	
